---
layout: post
title: "First Tech Post: thenorthernbunny site goes live!"
categories: [tech]
---

Decided to start a personal site. Figured I would learn some stuff by doing it, which I already have! It is to be a place to document personal projects I am working on. So with that, I will describe the process of getting the site going.

### The Architecture: Keeping Costs Low
First thing was to do some research to make this project as cheap as possible, since it's just a personal endeavor. I could have decided to use something like the free version of WordPress, Google Blogs, or Notion, but decided for now to register my domain with a registrar, use Cloudflare's free tier as my DNS, and host my actual pages on GitHub for free. So this website will cose 7 dollars for the first year, and 20 dollars per year after that.  

### 1. The Registrar (Namecheap)
To choose the registrar, I just did some comparative shopping and settled on Namecheap. So that is \$6.99 for 1 year, and will renew at maybe \$20 per year after that. Pretty good, and they give free WHOIS privacy protection, called WhoisGuard. Remember to push the slider for WhoisGuard to enabled for the domains you register. Disable things that they offer for a free trial, like custom DNS services, so that they don't auto-renew and charge later. All in all, it was a very simple process. I chose *The Northern Bunny* because I live in Canada, and I think bunnies are pretty cute. 

### 2. The DNS Layer (Cloudflare)
I chose Cloudflare to be the DNS for the flexibility and the local caching, so that people looking at the site in different areas would have local cached copies to speed up loading. Very pleased with the free tier service. Cloudflare seems to have a lot of cool services, need to investigate them more... maybe a future blog post! 

After creating your free tier account at cloudflare.com, you need to add your domain(s). Cloudflare will give you two nameserver addresses (like `xxx.ns.cloudflare.com`). Make sure that the proxy slider is enabled—it should be by default. Then you need to go back to Namecheap → Domain List → Manage → Nameservers → switch to Custom DNS and paste Cloudflare's nameservers in. Might take a little time to propagate, although it did not take much time for me. 

### 3. Hosting (GitHub Pages)
I already had a GitHub account, and they offer a free public website via GitHub Pages (`username.github.io`). So this will be my free hosting home :) 

You need to go to your GitHub account (or create one if you don't have one), then create a new public repo named exactly `yourdomain.github.io`... in my case, `thenorthernbunny.github.io`. 

In Cloudflare, you just configure two CNAME DNS records for your domain to point to your `reponame.github.io`, and an extra one to point `www.domain` to your GitHub repository. Since `https://thenorthernbunny.com` and `https://www.thenorthernbunny.com` are technically two different URLs, Cloudflare handles routing them cleanly, though modern web browsers usually strip out the `www` (that's a weird one I learned!). GitHub Pages includes free HTTPS, so you don't need to worry about buying that as an option from Namecheap. 

In summary: in Namecheap, I just configured my domain name `thenorthernbunny.com` to use the Cloudflare DNS servers, and set Cloudflare DNS to point to my public GitHub page with CNAME records. 

### 4. The Presentation (Jekyll & Hydejack)
Finally, I found a collection of themes designed specifically to jazz up your GitHub public page. I chose the **Jekyll Hydejack** theme. You just download it, make a repo, copy it to GitHub, and then configure it. Bit of a learning curve to figure stuff out there... at least for me, but I am getting there!
