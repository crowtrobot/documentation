# Getting Wildcard DNS working

## What's the goal?

Nextcloud's built-in collabra server seems to have stopped working. If I install that nextcloud app, I start getting gateway timeouts. So I might want to install a separate collabra VM. But that's going to need another subdomain, and that means some work to change the names in the certificate. So I've decided that instead I want a wildcard cert.

## Background

For many years my domain has had a few subdomains as CNAME records that point at another free dynamic DNS domain name. It might be time to change that.

So after a little bit of looking at the options, it looks like the dynamic DNS service in pfsense has an option that could update the DNS in hover, but it looks like that might not work with the 2 factor authentication on that account. And if it does, it looks like that's technically giving it access to everything, not just the DNS. So I'm not excited by that idea.

Instead it looks like I can move the DNS over to cloudflare. There I can make an API key that is only able to update the DNS records, and since it isn't also the registrar, if things come to the worst I can just switch the primary nameservers to someone else in hover, so this feels good.

I did one test domain, and it went well, so let's do the main one for real.

## Getting the DNS into cloudflare

For the sake of not putting all my eggs into one basket, I will leave one of the alternative spelling domain names with hover and just CNAME it to the one I will be moving, so if things get ugly with cloudflare some day I have a domain I can still get to and update.

So first step, make a cloudflare account, add the password to the password manager, turn on 2fa etc. The first guide I found for setting up dns seemed to assume you bought the domain name through cloudflare. So I didn't have the DNS option at first on my account. Turns out you have to first go to the top-right and click Add Site. So Add Site, put in the base of your domain (domain.com instead of bob.domain.com). You have to scroll down a bit to see the free option.

Just a brief sidebar at this point. Needing to scroll down for the free option isn't a big deal here. It's not exactly hidden, just not emphasized. They let me create an account with only an email verification. No need to have a credit card on file or something, so if I had accidentally selected a option with a cost, I would have found out long before getting charged. Absolutely no complaints from me, nothing but gratitude.

So anyway, choose free option, click continue. Here you make sure all the subdomains are right. It tries to pull them from the DNS record, but missed one for me. Also here I turned off the "proxy" option. I will come back and try that in the future but I don't want to risk it breaking things for now. KISS principal, make changes small with testing in between them.

Since the goal here is to make wildcards work, I added an A record for "@". This is the address for no subdomain (so the IP you get from google.com instead of www.google.com). I set this A name to point to 8.8.8.8 (for now, dynamic DNS will update it later, but this will let me see that the update is working). I also added a CNAME for \* to point to the bare domain (to that A name), which should let any subdomain just automatically work.

Then click continue, and it gives you some instructions of changing the domain's nameservers. This basically boiled down to going to hover, clicking "nameservers" at the top, selecting the domain, clicking edit, and then copy-and-paste in the ones from cloudflare. That's it, that starts pointing to the DNS records at cloudflare instead of the DNS records at hover. It might take a while for DNS caches to renew.

## Make an API token

Now we need pfsense to be able to update the @ record when the IP the ISP is giving us changes. To do that we are going to make an API key from pfsense that can authorize that change, but nothing else. Click the little meeple icon in the top-right corner, and click "my profile" Here we will click the API Tokens tab on the left, and click create token. Select the "edit zone dns" token template,

Click the little pencil next to the token name, and set a name that will help you recognize this token later, something like "Dynamic DNS on pfsense".

<figure><img src=".gitbook/assets/image (1) (1).png" alt=""><figcaption><p>API token screen from CloudFlare</p></figcaption></figure>

In the top section, "zone" "DNS" and "Edit" will already be selected, so leave that. In the next section we will choose "Include" "specific zone" and then need to chose our domain here. I actually have 2 domains now getting DNS from cloudflare, so I will also click "add more" so I have a row for each domain. This way if I add more, they aren't automatically covered by this API token.

I wish I could use the client IP filtering, but if I knew what IPs I would be using from now on I wouldn't need the dynamic DNS, so I will leave that one alone. I will also leave the TTL alone since I don't really feel like I need to rotate my API keys periodically (but if I were doing this for an enterprise I think I would. It a good excuse to force there to be a policy for manually updating these things routinely so as employees come and go you know someone has access and experience).

Don't save this API token in a password manager or anything like that. If you ever need to recover it, you are better off deleting it and making a new one.

## Setup the Dynamic DNS client in PFsense

Now go to pfesne and log in. Go to Service > Dynamic DNS and click Add. Choose cloudflare as the service type, select wan as the interface we are interested in, hostname @, and put in the domain name. Leave the username blank (I'm not sure that's what they meant with the ambiguous hint to use the zoneid, but it works, and putting the domain name in there did not). Paste the token into the password field and the confirm password field.

Check the DNS stuff in cloudflare to make sure that the 8.8.8.8 IP was replaced, just to be sure it is all working.
