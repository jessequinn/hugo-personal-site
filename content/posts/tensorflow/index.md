---
title: 'Compiling Tensorflow'
description: 'A brief run through for compiling a pip wheel for an optimized gpu Tensorflow'
cover:
  image: 'solen-feyissa-IfWFKG3FXE4-unsplash.jpg'
  alt: 'Compiling Tensorflow'
date: "2019-10-28"
ShowToc: true
---

Although the majority of the information can be found at [Tensorflow](https://www.tensorflow.org/install/source), I will provide a short Bash script that does all the thinking for you. Because I use pyenv virtualenv and the lastest cuda/cudnn on Manjaro, I ran into an issue with the prebuilt pip tensorflow-gpu. Of couse it complained about the cuda version not being 10.0 while I have 10.1 installed. To get around this, atleast from what I know, you will need to recompile Tensorflow... be prepared.. it takes a long time even for an i7 with 65gb ramm!!! Of course, you can downgrade, but I do not like doing that. Before you begin, you will need to do a 

	pip install wheel 

Using portions of Archlinux [PKGBUILD](https://www.archlinux.org/packages/community/x86_64/python-tensorflow-opt-cuda/) for tensorflow, I decided to make a Bash script that would do all the thinking for me, as I simply wanted to create a pip wheel for my virtualenv!

<script src="https://gist.github.com/jessequinn/724859b576aea1efbfa20bd28d006cb2.js"></script>

Modify as you desire and remember this will take sometime.

In addition, you will need a patch,

<script src="https://gist.github.com/jessequinn/d622364a22805d8f892756d47ff6519d.js"></script>
<script src="https://gist.github.com/jessequinn/34ea874fe8ec4e7bef8de8fa6ba6aba8.js"></script>

Best of luck
