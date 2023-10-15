---
layout: post
title: "Deploying a Jekyll Site Using AWS Amplify"
date: "2022-10-17"
post_image: /assets/images/joshua-fuller-p8w7krXVY1k-unsplash.jpg
tags: [ruby, jekyll, aws, amplify]
categories: [devops]
author: jose
comments: true
dark_header: false
---

## TL;DR

In order to deploy a site built using [Jekyll](https://jekyllrb.com/) version `4.2`, I had to create a custom [Ubuntu-based build image](https://github.com/AptDevCode/aws-amplify-build-image) to run on [AWS Amplify](https://docs.aws.amazon.com/amplify/latest/userguide/welcome.html). The custom build image would use [rbenv](https://github.com/rbenv/rbenv) to install Ruby version `2.7.1`.

## The Problem

One of my favorite tools to use for static websites is AWS Amplify. Before Amplify, the task of deploying a website on AWS was a bit more tedious: setting up S3 buckets, adding entries into Route 53, and fiddling with CloudFront to acquire the SSL Cert. AWS Amplify does all that for you under the hood. It will detect a commit to a branch you specify, start a build, and then wave a magic wand to deploy the artifacts and make them accessible via a public URL. So it reduces all the complexity of setup and deployment to a simple commit!

I normally use [Jekyll](https://jekyllrb.com/) to build static sites. For [AptDev](https://aptdev.online), I pulled a nice looking template from the web to start development. I had to tinker a bit with the template, as it was not designed according to Jekyll standards. Once I finished testing it locally and was satisfied with the results, I started the process of setting up an Amplify app to deploy it.

Amplify has Jekyll/Ruby support out of the box. I had used it before for other sites. So imagine my surprise when the first build failed with this error

```bash
2022-09-25T00:19:45.736Z [WARNING]: jekyll-feed-0.16.0 requires ruby version >= 2.5.0, which is incompatible with the current version, ruby 2.4.6p354
```

Unfortunately, The template came with some dependencies that required a higher version of Ruby.

I attempted to use the [Live Package Override](https://docs.aws.amazon.com/amplify/latest/userguide/custom-build-image.html#setup-live-updates) feature, but that also failed with the same error.

<figure class="figure">
  <img
    src="{{site.baseurl}}/assets/images/screenshots/2022-10-17_screenshot1.png"
    alt="blog img"
  />
  <figcaption class="figure-caption text-center">
    Live Package Version Override
  </figcaption>
</figure>

{% include amazon_ad.html %}

## The Solution

So I had no other choice but to use a custom build image. I polished my laptop screen, brewed some Cafe Bustelo, and hunkered down for a long ride ☕. Using the power of Docker, AWS documentation, and Google search, I went through many iterations to create the one custom image to build them all!

- Ultimately, I settled on using an ubuntu based image to install rbenv.
- Within the image, I used rbenv to install the required Ruby version, which was `2.7.1`
- Along with rbenv, I also installed the required packages needed to support Ruby and Jekyll version `4.2`, along with some `g++` tools for the `eventmachine` dependency.
- I had to modify the `amplify.yaml` file to properly load rbenv on startup

You can find more details about the solution on this [GitHub issues thread](https://github.com/aws-amplify/amplify-hosting/issues/2565#issuecomment-1279684405).

The Docker file defining the image can be viewed [here](https://github.com/AptDevCode/aws-amplify-build-image/blob/develop/amplify-ruby-build/Dockerfile). If you wish to use the image yourself, you can pull it from a [Elastic Container Registry](https://gallery.ecr.aws/z8u1m2l7/aptdev/amplify-ruby-build).

```yaml
version: 1
env:
  variables:
    JEKYLL_ENV: production
frontend:
  phases:
    preBuild:
      commands:
        - eval "$(/root/.rbenv/bin/rbenv init - bash)"
        - rbenv global 2.7.1
        - gem install bundler
        - bundle install
    build:
      commands:
        - bundle exec jekyll build
  artifacts:
    baseDirectory: _site
    files:
      - "**/*"
  cache:
    paths: []
```

## The Challenges

No effort is completed without overcoming some hurdles. Here were the challenges I faced when creating the image:

1. **Amazon Linux** - At first, I tried basing my image off of Amazon Linux. The supported packages are very limited. Ultimately it did not support the required `g++` dependency I needed to compile one of the gems, so I ended switching to Ubuntu instead.
1. **Package Dependencies** - It took several tries to figure out all the right packages. Ultimately, moving to Ubuntu reduced those dependencies, since the base image came with them already installed.
1. **Trial and Error** - For some reason, AWS Amplify didn't honor the `~/.bashrc` file. By default it runs as root, so when I installed rbenv, I assumed installing as root would be fine. I ended up modifying the `amplify.yaml` file to ensure rbenv loaded on build.

## Conclusion

I learned a lot about using custom build images on AWS Amplify. I also learned more about Ruby and Jekyll via rbenv.

I love AWS Amplify. It makes deploying websites so much easier. Perhaps in a future post, I'll show you how to do the same using Google Cloud and Firebase.

If you're also facing issues deploying your Jekyll site via AWS Amplify, please feel free to use my "One Custom Image to Build Them All"!

Photo by [Joshua Fuller](https://unsplash.com/@joshuafuller?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/ruby?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
