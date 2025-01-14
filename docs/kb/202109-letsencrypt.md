---
id: kb-202109-letsencrypt
title: Let's Encrypt DST Root CA X3 expiry Sept 30th 2021 
---

# Summary

Certificate trust relies on the "root" issuing certificate being trusted by your computer. 

The root certificate issues an Intermediate certificate which in turn is used to issue general certificates such as the ones for your website. This is called a "Chain" of trust. Your certificate (called a Leaf or end-entity certificate) will be validated by following this chain.

*From Sept 30th 2021 Let's Encrypts previous root certificate *DST Root CA X3* (and it's R3 intermediate) will expire. It has been replaced by their *ISRG Root X1* certificate (and replacement R3 intermediate).*

:::note

**In most cases, your systems will automatically switch over to the next trusted chain they can find.**

:::


# Possible issues
In some cases, the expiry of the root (and its related expiring R3 intermediate certificate) may causes certificates to be considered untrusted or invalid. To fix this you need to make your server use (serve) the correct chain.

In other cases, the issue may be with the client computer.

The version of the *R3* intermediate signing certificate which chains to *DST Root CA X3* expires **September 29 19:21:40 2021 GMT.**

The *DST Root CA X3* root certificate expires **September 30 14:01:15 2021 GMT**.

# Solutions

## Servers
The following solutions mainly apply to Windows servers running IIS or other windows based services which use the windows trust store. Apache, nginx etc have their own trust mechanisms :

### Servers with problems after expiry
:::tip Check Your Chain
1 - To diagnose a chain issue for your server, scan one of your webservers domains with a [chain checker](#chain-checking)

2 - If your chain contains the expired R3 after it's expiry, reboot your server to clear cached chains.

3 - If the chain issue persists, re-request your certificate in Certify The Web to force a binding refresh or choose `Certificate > Advanced > Actions > Re-apply Certificate To Bindings`.
:::

### Server fixes and how to switch chain before (or after) expiry

We recommend the following steps to initially correct your servers certificate chain:

1. Install the latest version of *Certify The Web* from https://certifytheweb.com or use the in-app update process. Even if you are not using this app currently, the upgrade will automatically fix common trust store issues (and it can then be uninstalled if not being used).
2. **Reboot your server** (this forces windows to re-evaluate the served certificate chains). You may be able to avoid this reboot by using `iisreset /restart` to just restart IIS.
3. Check your certificate chain with a [chain checker](#chain-checking)
4. In some cases you may need to refresh your IIS bindings, the easiest method is to click 'Request Certificate' to change certificate and update bindings.

:::tip Valid Chains
The default chain served by Windows (depending on the state of your server trust store) will be either:

#### **Chain 1 (modern)** : (your cert) > R3 > ISRG Root X1
This chain is supported by current operating systems

#### **Chain 2 (legacy)** : (your cert) > R3 > ISRG Root X1 > DST Root CA X3
This chain is ideal if you need broader compatibility with older operating systems, including Android 7.0 and lower.
:::

:::warning Legacy Chain Warning 2021/09/30
In our recent testing we have found that when `DST Root CA X3` expires, although Windows can initially serve the legacy chain it will revert to the modern chain automatically when it notices `DST Root CA X3` has expired. This will impact serving content to old versions of Android and some other older operating systems which don't trust `ISRG Root X1`. If you require legacy support consider changing [Certificate Authority](/docs/guides/certificate-authorities).
:::

For IIS etc, you can only serve one of these chains per Windows server (machine), not a combination per site etc. The default trust store maintenance in Certify The Web will provide the *modern* chain. If you need the legacy chain you may still need import the cross signed ISRG Root X1 (see *Switching to Chain 2*, below) unless it was already installed.

Either chain is fine for most purposes (see above warning). 

:::danger If your chain is still: (your cert) > R3 > DST Root CA X3
If you still see this old chain after the DST Root CA X3 expiry (after updating Certify The Web and after rebooting), then you need to resolve this urgently. At a minimum you must ensure ISRG Root X1 (Self signed) is installed under your machine Trusted Certification Authorities using certlm.msc and remove the R3 issued by DST Root CA X3 from Intermediate Certification Authorities. See the [further troubleshooting](#further-troubleshooting) section below.
:::

### Switching to Chain 1 (modern)
If you are serving the *ISRG Root X1 > DST Root CA X3 (self signed)* chain and want to switch to just the *ISRG Root X1* chain:
- remove the cross signed version of the ISRG Root X1 (issued by DST Root CA X3) from your machine trust store. 
- You may need to reboot to see the effect.

### Switching to Chain 2 (legacy)
If you require compatibility with old versions of Android and other devices that don't know about *ISRG Root X1*, you need to serve **Chain 2**. To do so:
- install the version of the *ISRG Root X1* cert which is **cross signed (issued) by *DST Root CA X3***. https://letsencrypt.org/certs/isrg-root-x1-cross-signed.der into the *Trusted Certification Authorities* (or Intermediate Certification Authorities) machine store using certlm.msc. 
- You may need to reboot to see the effect.

If no other solution works or for any other reason you cannot update client trusts stores etc or require other broader compatibility, you may need to consider moving your certificate to a new Certificate Authority. Certify The Web supports a range of built-in [alternatives](/docs/guides/certificate-authorities). You could also alternatively use a front-end proxy service such as Caddy, nginx, Apache, or a hosted DNS proxy service like Cloudflare, but these require significant changes to implement.

## Non-IIS servers (Apache, nginx etc on Windows or Linux)
Verify that your service is configured to use your certificate, with it's private key *and* it's **chain**. These services will work without pointing to a chain file but in the case of the expired R3 your clients will try to resolve the R3 themselves (because you haven't given it to them) and they may then resolve it to the old (expired) one.

## Clients (browsers etc)
If your site is working for most devices but not for some, the problem is with their trust store (their list of trusted root certificate).

### Windows PCs
On windows PCs, simply browsing to a website using Chrome, Edge etc with updated the client trust store with the required certificates. Browsing to https://valid-isrgrootx1.letsencrypt.org/ will prompt Windows to include *ISRG Root X1* in its trust store automatically.

### macOS, iOS etc
Some operating systems hold onto the expired `R3 > DST Root CA X3` chain even if your server is no longer using it. Try a restart of the affected client device.

## Java based systems etc
Some applications maintain their own trust store. You may need to add the newer ISRG Root X1 certificate into your systems trusts store. Any system that can't be updated needs to see the legacy chain or you need to switch CA.

e.g. for Java you might use: `keytool -import -alias isrgrootx1 -keystore $JAVA_HOME/jre/lib/security/cacerts -trustcacerts -file isrgrootx1.cer` ([credit](https://community.letsencrypt.org/t/help-thread-for-dst-root-ca-x3-expiration-september-2021/149190/258))

## Other considerations
### Export Tasks
If you use Certify The Web to export certificates to pem files etc (for Apache or other servers), the chain you get in the export will correspond which the chain your server is currently building. The "Preferred Issuer" setting for the certificate authority will have *no effect*, because Windows is overriding the chain.

## Further Troubleshooting

- A registry method to delete the old R3 is [documented here](https://gist.github.com/skusiak/83db2ba2fc1804b89151db01e97bbbec)
- If your expired chain keeps coming back, moved (or install) the expired `R3 issued by DST Root CA X3` into the Untrusted store using `certlm.msc` (Manage Computer Certificates).
- Further information and troubleshooting steps are here: https://community.certifytheweb.com/t/upcoming-expiry-of-dst-root-ca-x3-and-r3-intermediate-for-lets-encrypt/1480

:::tip Licensed Users can contact Certify The Web support
While the problem itself relates to and is controlled by Windows and Let's Encrypt, licensed users can contact Certify The Web support via `support at certifytheweb.com` if they have further related questions and issues they need advice with. We are in the AEST (Australia) Time Zone. A higher than normal volume of support tickets are expected immediately up to, during and after the root expiry so please expect delays and perform as much of your own troubleshooting and research as you can.
:::

## Useful Information
Visit the Let's Encrypt support community for more information about the root expiry and chain changes: https://community.letsencrypt.org/t/production-chain-changes/150739/4
### Chain Checking
Other ways to check and diagnose chain issues:
- Qualsys SSL Server Test: https://www.ssllabs.com/ssltest/ is useful for full diagnostics of your certificate chain.
- Namecheap SSL Checker: https://decoder.link/sslchecker/
- OpenSSL: `openssl s_client -connectyour.domain.com:443 -servername  your.domain.com`

