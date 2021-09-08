---
title:
description:
cover:
  image: 'david-thielen-Om-Z4TDfv7o-unsplash.jpg'
  alt: ''
date: "2021-09-07"
ShowToc: true
---

## Introduction
With the recent release of Grafana 8 a new opt-in feature for alerting has 
been made [available](https://grafana.com/docs/grafana/latest/alerting/unified-alerting/). 
[Grafana](https://grafana.com/docs/grafana/latest/http_api/alerting/)
alerting has dramatically changed, and in my opinion, for the good. Why? Primarily due to the fact you are no longer 
limited to a dashboard. Alerts, rules, can be made directly in the alert manager tab.

With this change in Grafana I was hoping to find a GO project that would consist of a REST API wrapper for this new feature; however, 
I could not find anything. Therefore, I decided to quickly write a small program to print current alerts. 

## Code
To begin, I wrote a simple `main.go` to read a `config.yaml` that contains that Grafana host and apikey information. This feature utilizes [viper](https://github.com/spf13/viper) package.
I then create an HTTP client that interacts or rather calls the REST API endpoints of the Grafana server. In this case, to retrieve all alerts:

```go
// main.go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/inancgumus/screen"
	log "github.com/sirupsen/logrus"
	"github.com/spf13/viper"

	"grafana-golang-sdk/pkg/grafana"
)

func main() {
	viper.SetConfigName("config")
	viper.SetConfigType("yaml")
	viper.AddConfigPath(".")
	err := viper.ReadInConfig()
	if err != nil {
		log.Fatal(err)
	}
	host := viper.GetString("host")
	apikey := viper.GetString("apikey")
	if host != "" && apikey != "" {
		c, err := grafana.NewClient(host, apikey, grafana.DefaultHTTPClient)
		if err != nil {
			log.Fatal(err)
		}
		alerts, err := c.GetAllAlerts(context.TODO())
		if err != nil {
			panic(err)
		} else {
			for t := range time.Tick(10 * time.Second) {
				screen.Clear()
				screen.MoveTopLeft()
				fmt.Println("")
				fmt.Println("")
				fmt.Println("")
				go alertSleep(t, alerts)
			}
		}
	}
}

func alertSleep(tick time.Time, alerts []grafana.Alert) {
	for _, v := range alerts {
		log.WithFields(log.Fields{
			"status":      v.Status.State,
			"severity":    v.Labels.Severity,
			"description": v.Annotations.Description,
			"tick":        tick.Format("15:04:05.000"),
		}).Info(v.Annotations.Summary)
	}
}
```

I am a big fan of the [logrus](https://github.com/sirupsen/logrus) package, and I commonly employ the package in all my projects. I use a `time.Tick`
 set at 10 seconds to call the `alertSleep` function that prints all current/firing alerts.

The heart of the project is found in the `connection.go`. The file should be self-explanatory; however, for brevity the most important portion is the `type Alert struct`, which 
is used to convert/unmarshal the JSON response from Grafana. A very useful website, [JSON-to-Go](https://mholt.github.io/json-to-go/), quickly converts JSON to a usable struct in GO.

```go
// pkg/grafana/connection.go
package grafana

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"net/url"
	"path"
	"strings"
	"time"
)

// DefaultHTTPClient for monkey patching
var DefaultHTTPClient = http.DefaultClient

// errors
var (
	errBasicAuth = "basic auth not allowed"
)

// Client uses Grafana REST API for interacting with Grafana server.
type Client struct {
	baseURL       string
	authorization string
	client        *http.Client
	// get specific function for get query type.
	get func(context.Context, *http.Client, string, string, string, url.Values) ([]byte, int, error)
}

// NewClient initializes Client for interacting with an instance of Grafana server.
func NewClient(host, token string, client *http.Client) (*Client, error) {
	baseURL, err := url.Parse(host)
	if err != nil {
		return nil, err
	}
	if !strings.Contains(token, ":") {
		return &Client{
			baseURL:       baseURL.String(),
			authorization: fmt.Sprintf("Bearer %s", token),
			client:        client,
			get: func(ctx context.Context, client *http.Client, baseURL, authorization, query string, params url.Values) ([]byte, int, error) {
				return request(ctx, client, baseURL, authorization, "GET", query, params, nil)
			},
		}, nil
	} else {
		return nil, errors.New(errBasicAuth)
	}
}

// Alert struct that represents the json response from the REST API endpoint.
type Alert struct {
	Annotations struct {
		ValueString string `json:"__value_string__"`
		Description string `json:"description"`
		Summary     string `json:"summary"`
	} `json:"annotations,omitempty"`
	EndsAt      time.Time `json:"endsAt"`
	Fingerprint string    `json:"fingerprint"`
	Receivers   []struct {
		Name string `json:"name"`
	} `json:"receivers"`
	StartsAt time.Time `json:"startsAt"`
	Status   struct {
		InhibitedBy []interface{} `json:"inhibitedBy"`
		SilencedBy  []interface{} `json:"silencedBy"`
		State       string        `json:"state"`
	} `json:"status"`
	UpdatedAt    time.Time `json:"updatedAt"`
	GeneratorURL string    `json:"generatorURL"`
	Labels       struct {
		AlertRuleUID             string `json:"__alert_rule_uid__"`
		Name                     string `json:"__name__"`
		AlertName                string `json:"alertname"`
		AppKubernetesIoInstance  string `json:"app_kubernetes_io_instance"`
		AppKubernetesIoManagedBy string `json:"app_kubernetes_io_managed_by"`
		AppKubernetesIoName      string `json:"app_kubernetes_io_name"`
		HelmShChart              string `json:"helm_sh_chart"`
		Instance                 string `json:"instance"`
		Job                      string `json:"job"`
		JobName                  string `json:"job_name"`
		KubernetesName           string `json:"kubernetes_name"`
		KubernetesNamespace      string `json:"kubernetes_namespace"`
		KubernetesNode           string `json:"kubernetes_node"`
		Namespace                string `json:"namespace"`
		Reason                   string `json:"reason"`
		Severity                 string `json:"severity"`
	} `json:"labels,omitempty"`
}

// GetAllAlerts gets all alerts.
func (c *Client) GetAllAlerts(ctx context.Context) ([]Alert, error) {
	var (
		raw  []byte
		an   []Alert
		code int
		err  error
	)
	if raw, code, err = c.get(ctx, c.client, c.baseURL, c.authorization, "api/alertmanager/grafana/api/v2/alerts", nil); err != nil {
		return nil, err
	}
	if code != 200 {
		return nil, fmt.Errorf("HTTP error %d: returns %s", code, raw)
	}
	err = json.Unmarshal(raw, &an)
	return an, err
}

type HttpClient interface {
	Do(req *http.Request) (*http.Response, error)
}

// request is a generic function for requests.
func request(ctx context.Context, client HttpClient, baseURL, authorization, method, query string, params url.Values, buf io.Reader) ([]byte, int, error) {
	u, _ := url.Parse(baseURL)
	u.Path = path.Join(u.Path, query)
	if params != nil {
		u.RawQuery = params.Encode()
	}
	req, err := http.NewRequest(method, u.String(), buf)
	if err != nil {
		return nil, 0, err
	}
	req = req.WithContext(ctx)
	req.Header.Set("Authorization", authorization)
	req.Header.Set("Accept", "application/json")
	req.Header.Set("Content-Type", "application/json")
	resp, err := client.Do(req)
	if err != nil {
		return nil, 0, err
	}
	data, err := ioutil.ReadAll(resp.Body)
	resp.Body.Close()
	return data, resp.StatusCode, err
}
```

Don't forget unit testing:

```go
// pkg/grafana/connection_test.go
package grafana

import (
	"context"
	"io"
	"net/http"
	"net/url"
	"reflect"
	"strings"
	"testing"
)

func TestNewClient(t *testing.T) {
	type args struct {
		host   string
		token  string
		client *http.Client
	}
	tests := []struct {
		name    string
		args    args
		want    *Client
		wantErr bool
	}{
		{
			name: "test",
			args: args{
				host:   "http://localhost",
				token:  "test",
				client: http.DefaultClient,
			},
			want: &Client{
				baseURL:       "http://localhost",
				authorization: "Bearer test",
				client:        http.DefaultClient,
			},
			wantErr: false,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := NewClient(tt.args.host, tt.args.token, tt.args.client)
			if (err != nil) != tt.wantErr {
				t.Errorf("NewClient() error = %v, wantErr %v", err, tt.wantErr)
				return
			}
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("NewClient() = %v, want %v", got, tt.want)
			}
		})
	}
}

func TestClient_GetAllAlerts(t *testing.T) {
	type args struct {
		ctx context.Context
	}
	tests := []struct {
		name    string
		c       *Client
		args    args
		want    []Alert
		wantErr bool
	}{
		{
			name: "test",
			c: &Client{
				baseURL:       "http://localhost",
				authorization: "Bearer test",
				client:        nil,
				get: func(ctx context.Context, client *http.Client, s string, s2 string, s3 string, values url.Values) ([]byte, int, error) {
					return []byte(`[
  {
    "annotations": {
      "__value_string__": "test",
      "description": "test",
      "summary": "test"
    },
    "status": {
      "state": "active"
    },
    "labels": {
      "severity": "warning"
    }
  }
]`), 200, nil
				},
			},
			args: args{context.TODO()},
			want: []Alert{{
				Annotations: struct {
					ValueString string `json:"__value_string__"`
					Description string `json:"description"`
					Summary     string `json:"summary"`
				}{"test", "test", "test"},
				Fingerprint: "",
				Receivers:   nil,
				Status: struct {
					InhibitedBy []interface{} `json:"inhibitedBy"`
					SilencedBy  []interface{} `json:"silencedBy"`
					State       string        `json:"state"`
				}{nil, nil, "active"},
				GeneratorURL: "",
				Labels: struct {
					AlertRuleUID             string `json:"__alert_rule_uid__"`
					Name                     string `json:"__name__"`
					AlertName                string `json:"alertname"`
					AppKubernetesIoInstance  string `json:"app_kubernetes_io_instance"`
					AppKubernetesIoManagedBy string `json:"app_kubernetes_io_managed_by"`
					AppKubernetesIoName      string `json:"app_kubernetes_io_name"`
					HelmShChart              string `json:"helm_sh_chart"`
					Instance                 string `json:"instance"`
					Job                      string `json:"job"`
					JobName                  string `json:"job_name"`
					KubernetesName           string `json:"kubernetes_name"`
					KubernetesNamespace      string `json:"kubernetes_namespace"`
					KubernetesNode           string `json:"kubernetes_node"`
					Namespace                string `json:"namespace"`
					Reason                   string `json:"reason"`
					Severity                 string `json:"severity"`
				}{
					AlertRuleUID:             "",
					Name:                     "",
					AlertName:                "",
					AppKubernetesIoInstance:  "",
					AppKubernetesIoManagedBy: "",
					AppKubernetesIoName:      "",
					HelmShChart:              "",
					Instance:                 "",
					Job:                      "",
					JobName:                  "",
					KubernetesName:           "",
					KubernetesNamespace:      "",
					KubernetesNode:           "",
					Namespace:                "",
					Reason:                   "",
					Severity:                 "warning",
				},
			}},
			wantErr: false,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := tt.c.GetAllAlerts(tt.args.ctx)
			if (err != nil) != tt.wantErr {
				t.Errorf("Client.GetAllAlerts() error = %v, wantErr %v", err, tt.wantErr)
				return
			}
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("Client.GetAllAlerts() = %v, want %v", got, tt.want)
			}
		})
	}
}

type ClientMock struct{}

func (c *ClientMock) Do(*http.Request) (*http.Response, error) {
	r := io.NopCloser(strings.NewReader("test"))
	return &http.Response{
		Status:           "OK",
		StatusCode:       200,
		Proto:            "",
		ProtoMajor:       0,
		ProtoMinor:       0,
		Header:           http.Header{},
		Body:             r,
		ContentLength:    0,
		TransferEncoding: nil,
		Close:            false,
		Uncompressed:     false,
		Trailer:          nil,
		Request:          nil,
		TLS:              nil,
	}, nil
}

func Test_request(t *testing.T) {
	type args struct {
		ctx           context.Context
		client        HttpClient
		baseURL       string
		authorization string
		method        string
		query         string
		params        url.Values
		buf           io.Reader
	}
	tests := []struct {
		name    string
		args    args
		want    []byte
		want1   int
		wantErr bool
	}{
		{
			name: "test",
			args: args{
				ctx:           context.TODO(),
				client:        &ClientMock{},
				baseURL:       "test",
				authorization: "test",
				method:        "GET",
				query:         "test",
				params:        nil,
				buf:           nil,
			},
			want:    []byte(`test`),
			want1:   200,
			wantErr: false,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, got1, err := request(tt.args.ctx, tt.args.client, tt.args.baseURL, tt.args.authorization, tt.args.method, tt.args.query, tt.args.params, tt.args.buf)
			if (err != nil) != tt.wantErr {
				t.Errorf("request() error = %v, wantErr %v", err, tt.wantErr)
				return
			}
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("request() got = %v, want %v", got, tt.want)
			}
			if got1 != tt.want1 {
				t.Errorf("request() got1 = %v, want %v", got1, tt.want1)
			}
		})
	}
}
```

Coverage is about 35%. Therefore, lots of room for improvement; however, some basic testing was required.

## Final Words
A super simple GO app to watch alerts on Grafana 8.1. If you have a suggestion, I suppose use the `change me` link above.