---
layout: post
title: "First Tech Post: thenorthernbunny site goes live!"
categories: [tech]
---

Decided to start a personal site. Figured I would learn some stuff by doing it, which I already have! It is to be a place to document personal projects I am working on. So with that, I will describe the process of getting the site going.

### The Architecture: Keeping Costs Low
First thing was to do some research to make this project as cheap as possible, since it's just a personal endeavor. I could have decided to use something like the free version of WordPress, Google Blogs, or Notion, but decided for now to register my domain with a registrar, use Cloudflare's free tier as my DNS, and host my actual pages on GitHub for free. 

### 1. The Registrar (Namecheap)
To choose the registrar, I just did some comparative shopping and settled on Namecheap. So that is \$6.99 for 1 year, and will renew at maybe \$20 per year after that. Pretty good, and they give a free WHOIS privacy protection. Just remember to disable things that they offer for free trial, like custom DNS services, so that they don't auto-renew and charge later. All in all, it was a very simple process. I chose *The Northern Bunny* because I live in Canada, and I think bunnies are pretty cute. 

### 2. The DNS Layer (Cloudflare)
I chose Cloudflare to be the DNS for the flexibility and the local caching, so that people looking at the site in different areas would have local cached copies to speed up loading. Very pleased with the free tier service. Cloudflare seems to have a lot of cool services, need to investigate them more... maybe a future blog post! 

### 3. Hosting (GitHub Pages)
I already had a GitHub account, and they offer a free public website via GitHub Pages (`username.github.io`). So in Namecheap, I just configured my domain name `thenorthernbunny.com` to use the Cloudflare DNS servers, and set Cloudflare to point to my public GitHub page.

### 4. The Presentation (Jekyll & Hydejack)
Finally, I found a collection of themes designed specifically to jazz up your GitHub public page. I chose the **Jekyll Hydejack** theme. You just download it, make a repo, copy it to GitHub, and then configure it. Bit of a learning curve to figure stuff out there... at least for me, but I am getting there!
