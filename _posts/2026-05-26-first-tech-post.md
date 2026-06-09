---
layout: post
title: "First Tech Post: thenorthernbunny site goes live!"
categories: [tech]
---

Decided to start a personal site! I figured I would learn some new things by doing it from scratch—and I already have. This site will serve as a place to document the personal projects I am working on. To kick things off, I will describe the exact process of getting this website up and running.

### The Architecture: Keeping Costs Low

The first step was to do some research to make this project as budget-friendly as possible, since it's just a personal endeavor. I could have used the free tier of WordPress, Blogger, or Notion, but I decided instead to register a custom domain, use Cloudflare's free tier for DNS, and host the actual pages on GitHub for free. 

Ultimately, this website will cost just $7 for the first year, and roughly $20 per year after that. 

> **The Content Pipeline:**
> Write posts in **Markdown** (plain text files)
> $\downarrow$
> Push files to the **GitHub repository** (`NorthernBunny.github.io`)
> $\downarrow$
> **GitHub Pages** builds the site using **Jekyll** automatically
> $\downarrow$
> Jekyll applies the styled **Hydejack theme**
> $\downarrow$
> Cloudflare handles **DNS, security, and HTTPS** caching
> $\downarrow$
> Site is live and secure at **thenorthernbunny.com**

---

### 1. The Registrar (Namecheap)

To choose the registrar, I did some comparative shopping and settled on **Namecheap**. It cost $6.99 for the first year and will renew at around $20 per year moving forward. They include free WHOIS privacy protection (WithMCM/WhoisGuard), which keeps your personal contact details off public databases. 

*Tip: When checking out, remember to toggle the privacy protection to enabled, and disable any paid free-trial add-ons (like custom DNS or email trials) so you don't get surprise auto-renewals later.*

I chose the name *The Northern Bunny* because I live in Canada, and honestly, bunnies are pretty cute. 

### 2. The DNS Layer (Cloudflare)

I chose **Cloudflare** to manage my DNS because of its flexibility, security, and global edge caching. Caching ensures that visitors from different regions load cached copies of the site from a nearby server, massively speeding up loading times. 

Setting it up is straightforward:
1. Create a free account at cloudflare.com and add your domain.
2. Cloudflare generates two custom nameservers (e.g., `xxx.ns.cloudflare.com`).
3. Log back into Namecheap, navigate to **Domain List** $\rightarrow$ **Manage** $\rightarrow$ **Nameservers**, switch the dropdown to **Custom DNS**, and paste Cloudflare's nameservers there. 
4. Ensure Cloudflare's proxy toggle (the orange cloud icon) is enabled for your traffic.

Propagation took almost no time for me, and the free tier offers an incredible amount of depth that I hope to explore in a future post.

### 3. Hosting (GitHub Pages)

I already had a GitHub account, and they offer brilliant free static hosting via **GitHub Pages**. 

To set this up, you need to create a public repository named exactly matching your username: `username.github.io` (in my case, `NorthernBunny.github.io`). 

Over in Cloudflare, you route traffic to GitHub by configuring DNS records:
* A **CNAME** record pointing the root domain (`thenorthernbunny.com`) to your GitHub Pages URL (`NorthernBunny.github.io`). Cloudflare uses a feature called "CNAME flattening" to make this work seamlessly on the root domain.
* A second **CNAME** record pointing `www` to your repository. 

GitHub Pages includes free managed SSL certificates, meaning you get fully secure HTTPS automatically without paying extra to a registrar.

### 4. The Presentation (Jekyll & Hydejack)

To make the raw text look beautiful, GitHub Pages natively integrates with **Jekyll**, a popular static site generator. It takes plain text files written in Markdown and compiles them into a complete HTML website. While modern alternatives like Hugo, Astro, or Eleventy exist, Jekyll works out of the box with GitHub Pages without needing external build actions.

For the visual design, I browsed [jekyll-themes.com](https://jekyll-themes.com/) and selected the **Hydejack** theme. 

Getting it running required a quick learning curve:
1. I downloaded the Hydejack starter kit from GitHub and uploaded the files to my repository.
2. I modified the `_config.yml` file to personalize the site title, tagline, description, and social links. 
3. Writing a post is as simple as creating a file named `YYYY-MM-DD-title.md` inside the `_posts` directory. 
4. Once committed and pushed to GitHub, an automated workflow builds and deploys the updates live in under a minute.

### 5. Final Notes & Future Upgrades

A few posts into this journey, I can comfortably reflect on the setup. Configuring the Hydejack theme took a little trial and error, as did finding my rhythm with Markdown formatting. However, I’ve now hit a groove where creating new posts is completely frictionless.

It is worth noting that GitHub's free tier places a **1GB limit** on repository sizes. If I start embedding heavy photography, media, or assets, I might hit that wall. Thankfully, workarounds are simple:
* **Image Offloading:** I could leverage a service like **Cloudinary** (which offers a generous 25GB free tier/bandwidth allocation) or an **Amazon S3 + CloudFront** setup.
* **Git LFS:** Use Git Large File Storage, which is integrated into GitHub and highly affordable.
* **Alternative Hosting:** I could easily migrate the backend deployment to **Netlify** or **Cloudflare Pages**. Netlify, for instance, offers relaxed repository size limitations and completely handles custom static site generation builds natively upon a git push. 

For now, GitHub Pages is working flawlessly. If it ever starts causing friction, I will migrate—and that will make for another great blog post!
