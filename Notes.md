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

## Key Takeaways

### Remember:
- **IP addresses** = Always belong to actual servers
- **Domain names** = Labels that point to servers
- **Each subdomain** = Needs separate DNS configuration
- **DNS treats** = `domain.com` and `www.domain.com` as different addresses
- **CNAME points to** = GitHub's domain (`alexmathai.github.io`), not your custom domain
- **A Record points to** = GitHub's IP addresses directly

### The Big Picture:
Your custom domain (`alexmathai.online`) is like putting your own sign on GitHub's building. Visitors see your sign, but they're still visiting GitHub's servers where your website files are actually stored.