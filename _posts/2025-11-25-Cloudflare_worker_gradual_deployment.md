---
title: "Traffic Splitting with Cloudflare Workers"
layout: single
excerpt: "Using Cloudflare Workers Gradual Deployments you can split traffic easily VERY CLOSE to your users"
sitemap: false
permalink: /split-traffic-with-cloudflare-workers
---

In my experience with traffic splitting I have worked with technologies like NGINX and Amazon Web Services (AWS) Application Load Balancers. But at my current company we use CloudFlare for many different tools and it's main benefit is it's proximity to the user. Cloudflare Workers and Rules lets us do SO MUCH very quickly without even reaching our infrastructure.

# TL;DR
Using Cloudflare Workers Gradual Deployments you can split traffic easily VERY CLOSE to your users.

# Table of Contents
<!-- TOC -->

- [TL;DR](#tldr)
- [Table of Contents](#table-of-contents)
- [Setup](#setup)
  - [Cloudflare Worker with multiple versions](#cloudflare-worker-with-multiple-versions)
  - [Header Transform Rule](#header-transform-rule)
- [Deploying your Gradual Deployment](#deploying-your-gradual-deployment)
  - [Step 1: Create a new worker](#step-1-create-a-new-worker)
  - [Step 2: Save multiple versions of that worker](#step-2-save-multiple-versions-of-that-worker)
  - [Step 3: Deploy both version with 0% and 100% traffic split to old](#step-3-deploy-both-version-with-0-and-100-traffic-split-to-old)
  - [Step 4: Add a worker route for your worker](#step-4-add-a-worker-route-for-your-worker)
  - [Step 5: Create your Header Transform Rule](#step-5-create-your-header-transform-rule)
  - [Step 6: Turn up your traffic split](#step-6-turn-up-your-traffic-split)
  - [(Optional) Step 7: Get Headers to target a specific worker version](#optional-step-7-get-headers-to-target-a-specific-worker-version)
- [Conclusion](#conclusion)
- [Takeaways](#takeaways)

<!-- /TOC -->

# Setup
The main two components that I leveraged to create this setup were
1. A **Cloudflare Worker** with multiple version
2. A **Header Transform Rule** to handle [Version Affinity]

The [Gradual Deployment Documentation] from Cloudflare mentions using a header `Cloudflare-Workers-Version-Key` to specify which version of the worker gets executed. I mistakingly put that in my worker code when it needs to exist before the worker code execution because that's when the worker version decision occurs. Thus number two (Header Transform Rule) is necessary.

## Cloudflare Worker with multiple versions
The Cloudflare Worker v0 for me looks something like this and simply passes through the worker to the original origin.
```js
/*
  Version 0 - simple passthrough to origin
*/

export default {
  async fetch(request) {
    return await fetch(request);
  }
};
```

The Cloudflare Worker v1 for me changes the origin because we are hosting on Vercel.
```js
/*
  This is version 1 - going to a vercel app
*/

const NEW_ORIGIN = "new-page.vercel.stage.awesome.com";

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    url.hostname = NEW_ORIGIN;
    const newRequest = new Request(url.toString(), request);
    return await fetch(newRequest);
  }
};
```
## Header Transform Rule
The header transform rule is used to get the users a consistent "version" of the worker. They have [documentation][Version Affinity] on this but it talks about a Rule Engine which to put it simply is just adding a **Request Header Transform** rule which occurs before the worker execution.

![Header Transform Rule Configuration](/images/post_images/header-transform-rule.webp)
# Deploying your Gradual Deployment
> TODO: terraform or wrangler deploy all the business.

I manually created all this worker because I didn't have the time to figure out how to automate with Github Actions the split deployments but I'm guessing there's a way.

## Step 1: Create a new worker
I chose to create a new hello world worker with the UI. Replace the code with the code from above that fetches the proper origins.

## Step 2: Save multiple versions of that worker
When in the edit code screen there's a **Deploy** button on the top right with a dropdown. If you click the dropdown arrow you can click **Save** instead of deploy. We don't really need to deploy these just yet so let's just save the worker.

**WRITE A GOOD COMMENT** this is visible when you actually deploy your versions.

THEN

edit the worker with your v1 code and save a new version.
## Step 3: Deploy both version with 0% and 100% traffic split to old
In the deployments tab of the worker you can initiate a new deploy.

![Cloudflare Worker Deployment Interface](/images/post_images/cloudflare-worker-deployment.webp)

## Step 4: Add a worker route for your worker
Worker Routes are annoying in a sense that they are exact. There's much documentation around this that you can find. But to put it simply you just need 1 worker route `https://the.test.com` in my case.

> Note: I had to create a second worker that catches all of my asset requests for my new vercel application and routes the traffic using something similar to my v1 script above.
## Step 5: Create your Header Transform Rule
Create a **Header Transform Rule** in your zone similar to the one described above. This will set `Cloudflare-Workers-Version-Key` based off a session cookie.

## Step 6: Turn up your traffic split
You can edit the traffic split by clicking on this **Update Deployment %** button. (Aren't you glad you put comments on your versions now 😘)

![Update Deployment Traffic Split](/images/post_images/update-deployment.webp)

## (Optional) Step 7: Get Headers to target a specific worker version
Now that you have deployed at least some of your traffic to this worker. You can mess with random cookies til you identify a cookie that targets whatever workers you want. In my case I was looking for vercel headers to get a session cookie that I could use to guarantee that I hit the v1 version of my worker.
```sh
# Example script to find my vercel deployment
for i in {1..100}; do
  uuid=$(uuidgen | tr '[:upper:]' '[:lower:]')
  echo "Testing: $uuid"
  curl -s -D - https://the.test.com -o /dev/null -H "Cookie: session=$uuid" | grep -i "X-Vercel" && echo "^^ Found Vercel for cookie: $uuid"
done
```

Then you can use that cookie to consistently get your v1.

# Conclusion
There are many paths to becoming a high level engineer. In the end it all comes down to **providing value in a scalable manner**. What that value is comes in many shapes and colors. You must operate with confidence, in a drivers before solutions manner to work together to build great software. We engineers are always learning and building these T-shaped skills as engineers and we need stay passionate about what we are working on to reach our full potential.

# Takeaways
* Gradual Deployments with Cloudflare workers is one way to split traffic to a given URL.
* You need to do your Version Affinity code separate in a **Header Request Transform** rule.

[Version Affinity]: https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments/#version-affinity
[Gradual Deployment Documentation]: https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments
