# Notes to understand - all the routing involved for my website

Clearly explains the difference between domains and subdomains.
And also explains the workflows of a user when they type the domain name vs when they type the subdomain name.

Also explains what **A records** are and what their function is.
Additionally explains what **CNAME records** are and what their function is.

# DNS Concepts: Complete Guide

## What is DNS?
**DNS (Domain Name System)** = The internet's phonebook that translates human-readable domain names into IP addresses that computers understand.

## IP Addresses: The Foundation
- **What**: Unique numerical address for every server/computer on the internet
- **Example**: `185.199.108.153`
- **Key Point**: IP addresses belong to physical servers (like GitHub's servers), NOT to domain names
- **Analogy**: Like a postal address for computers

## Domain Names vs IP Addresses
- **Domain Name**: Human-friendly label (like `alexmathai.online`)
- **IP Address**: Computer address (like `185.199.108.153`)
- **Relationship**: Domain names point to IP addresses through DNS records

## DNS Record Types

### A Record (Address Record)
- **Purpose**: Directly connects domain name → IP address
- **Example**: `alexmathai.online → 185.199.108.153`
- **What happens**: Direct connection to server's IP address

### CNAME Record (Canonical Name)
- **Purpose**: Points one domain name → another domain name (alias)
- **Example**: `www.alexmathai.online → alexmathai.github.io`
- **What happens**: 
  1. Looks up the alias (`alexmathai.github.io`)
  2. Finds that domain's IP address
  3. Connects to that IP

## Domain Hierarchy

### Structure Breakdown:
```
www.alexmathai.online
 ↑      ↑        ↑
sub   domain   TLD
domain  name
```

### Components:
- **TLD (Top Level Domain)**: `.online`, `.com`, `.org`
- **Domain Name**: `alexmathai` (what you buy from registrar)
- **Subdomain**: `www`, `blog`, `api` (created by you)

### Full Domain Examples:
- `alexmathai.online` ← Root/apex domain
- `www.alexmathai.online` ← Subdomain
- `blog.alexmathai.online` ← Subdomain
- `shop.alexmathai.online` ← Subdomain

## Custom Domains vs Default URLs

### GitHub Example:
- **Default GitHub URL**: `alexmathai.github.io` (free, but uses GitHub's domain)
- **Custom Domain**: `alexmathai.online` (your own domain, points to same GitHub servers)

### Why Custom Domains:
- Professional appearance
- Your own branding
- Same content, different address

## Why www and Root Domain are Different

### Key Concept:
`alexmathai.online` and `www.alexmathai.online` are **completely different addresses** to DNS.

### User Behavior:
- Some type: `alexmathai.online`
- Others type: `www.alexmathai.online`
- Both should work → Need separate DNS records

### Configuration Required:
```
A Record: alexmathai.online → 185.199.108.153
CNAME: www.alexmathai.online → alexmathai.github.io
```

## Alternative CNAME Configuration Options

### Option 1: CNAME → GitHub Domain (Standard)
```
A Record: alexmathai.com → 185.199.108.153
CNAME: www → alex-mathai-98.github.io
```
- **Pros**: GitHub's official recommendation, follows their documentation
- **Cons**: Slightly more complex routing path

### Option 2: CNAME → Your Custom Domain (Alternative)
```
A Record: alexmathai.com → 185.199.108.153  
CNAME: www → alexmathai.com
```
- **Pros**: More logical routing, easier to understand, common practice
- **Cons**: One additional DNS lookup step

### How Option 2 Works:
When user visits `www.alexmathai.com`:
1. **DNS Lookup**: "What's the IP for www.alexmathai.com?"
2. **DNS Response**: "It's an alias for alexmathai.com"
3. **Second Lookup**: "What's the IP for alexmathai.com?"
4. **DNS Response**: "185.199.108.153" (from A record)
5. **Connection**: Browser connects to GitHub's server

### Hub and Spoke Model:
Option 2 creates a centralized approach where all subdomains point to your root domain:
```
A Record: alexmathai.com → GitHub's IP
CNAME: www → alexmathai.com
CNAME: blog → alexmathai.com  
CNAME: shop → alexmathai.com
```

**Both options work perfectly** - choose based on preference or follow GitHub's standard recommendation.

## Real-World Flow Example

### When user visits `www.alexmathai.online`:
1. **DNS Lookup**: "What's the IP for www.alexmathai.online?"
2. **DNS Response**: "It's an alias for alexmathai.github.io"
3. **Second Lookup**: "What's the IP for alexmathai.github.io?"
4. **DNS Response**: "185.199.108.153" (GitHub's server)
5. **Connection**: Browser connects to GitHub's server
6. **Result**: GitHub serves your website content

### When user visits `alexmathai.online`:
1. **DNS Lookup**: "What's the IP for alexmathai.online?"
2. **DNS Response**: "185.199.108.153" (direct)
3. **Connection**: Browser connects to GitHub's server
4. **Result**: GitHub serves your website content

## How GitHub Knows Which Website to Serve

### The Two-Part Process:

#### Part 1: DNS A Records (Gets you to GitHub's servers)
- A records point your domain to GitHub's IP addresses
- **Example**: `alexmathai.com → 185.199.108.153`
- **Issue**: GitHub hosts MILLIONS of websites on these same servers

#### Part 2: GitHub Pages Setting (Tells GitHub which website to serve)
- Configure custom domain in GitHub repository settings
- Creates internal mapping: `alexmathai.com → your-username/your-repository`
- **Critical**: This step tells GitHub which repository to serve for your domain

### HTTP Host Header Magic:
When browser connects to GitHub's server, it sends:
```
Request to: 185.199.108.153
Host Header: alexmathai.com
```

GitHub receives this and checks its internal database:
> "alexmathai.com is configured for alex-mathai-98/portfolio-website repository"

### GitHub's Internal Mapping Table:
```
Domain: alexmathai.com → Repository: alex-mathai-98/portfolio-website
Domain: johnsmith.com → Repository: john-smith/blog  
Domain: sarahblog.net → Repository: sarah123/my-site
```

### Why Both Parts Are Essential:

#### A Records alone:
- Gets traffic to GitHub's servers
- **Problem**: GitHub doesn't know which website to serve

#### GitHub Pages setting alone:
- GitHub knows which repository belongs to your domain
- **Problem**: DNS doesn't know where to send traffic

#### Both together:
- ✅ DNS sends traffic to GitHub's servers
- ✅ GitHub serves the correct repository content
- ✅ Website works perfectly!

## Complete Routing Workflow

### When user types `alexmathai.com`:
1. **DNS Lookup**: "What's the IP for alexmathai.com?"
2. **DNS Response**: "185.199.108.153" (GitHub's server - direct from A record)
3. **Browser Connection**: Connects to GitHub's server
4. **Host Header**: Browser sends "Host: alexmathai.com"
5. **GitHub Lookup**: Checks internal mapping for alexmathai.com
6. **Repository Match**: Finds alex-mathai-98/portfolio-website
7. **Content Served**: GitHub serves your website content
8. **User Sees**: Your website at alexmathai.com

### When user types `www.alexmathai.com`:
1. **DNS Lookup**: "What's the IP for www.alexmathai.com?"
2. **DNS Response**: "It's an alias for alex-mathai-98.github.io" (from CNAME)
3. **Second Lookup**: "What's the IP for alex-mathai-98.github.io?"
4. **DNS Response**: "185.199.108.153" (GitHub's server)
5. **Browser Connection**: Connects to GitHub's server
6. **Host Header**: Browser sends "Host: www.alexmathai.com"
7. **GitHub Lookup**: Checks internal mapping for www.alexmathai.com
8. **Repository Match**: Finds alex-mathai-98/portfolio-website
9. **Content Served**: GitHub serves your website content
10. **User Sees**: Your website at www.alexmathai.com

## Key Takeaways

### Remember:
- **IP addresses** = Always belong to actual servers
- **Domain names** = Labels that point to servers
- **Each subdomain** = Needs separate DNS configuration
- **DNS treats** = `domain.com` and `www.domain.com` as different addresses
- **CNAME points to** = GitHub's domain (`alexmathai.github.io`), not your custom domain
- **A Record points to** = GitHub's IP addresses directly
- **GitHub Pages setting** = Creates the crucial mapping between your domain and repository
- **Host Header** = How GitHub knows which website to serve from millions on the same servers

### The Complete Picture:
Your custom domain setup requires **THREE components**:
1. **DNS A Records**: Point your domain to GitHub's servers
2. **DNS CNAME Record**: Handle www subdomain routing
3. **GitHub Pages Configuration**: Tell GitHub which repository to serve for your domain

It's like a relay race: DNS gets the request to GitHub's servers, then GitHub's internal system takes over to serve the right content based on your Pages configuration.

# Instructions to locally develop and test the website

```bash
# Activate conda environment and start development server
source ~/miniconda3/etc/profile.d/conda.sh && conda activate jekyll-site && bash tools/run.sh
```

This will start the Jekyll development server at `http://127.0.0.1:4000/portfolio-website/` with live reload enabled.