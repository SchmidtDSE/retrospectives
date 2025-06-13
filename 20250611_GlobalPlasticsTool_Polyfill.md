# Polyfill.io Supply Chain Attack

**Summary**: This retrospective documents a near-miss security incident involving the compromise of polyfill.io, a third-party JavaScript polyfill service that was integrated into our application in 2023. While our enhanced privacy and security practices likely protected us from the attack, the same factors which likely safeguarded our software also highlight areas for improvement in our supply chain security monitoring. This focuses on the Global Plastics AI Policy Tool.

## Incident Overview

**Projects**: [Global Plastics AI Policy Tool](https://global-plastics-tool.org/)

**Responders**: A Samuel Pottinger (@sampottinger)

**Report / CVE**: [Sansec-2024-06-25](https://sansec.io/research/polyfill-supply-chain-attack) (See related [CVE-2024-38526](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-38526)).

**Attack Timeline**: The polyfill.io domain was compromised by a Chinese threat actor group, with malicious code injection beginning around June 25, 2024. [Security researchers detected the compromise](https://www.bleepingcomputer.com/news/security/polyfillio-javascript-supply-chain-attack-impacts-over-100k-sites/) and the domain was taken offline by Namecheap on June 28, 2024.

**Discovery Date**: June 10, 2025 during routine third-party resource audit.

**Impact**: Currently we believe that there was no compromise of our systems or users.

**Status**: Incident closed - no evidence of compromise but preventive measures implemented.

## Technical Details
This supply chain attack relied on injecting malicious code and impacted [hundreds of thousands of hosts including government services](https://censys.com/blog/july-2-polyfill-io-supply-chain-attack-digging-into-the-web-of-compromised-domains). We beelive were not impacted due to privacy protections we put in place for our users but these same measures also prevented our discovery of this near-miss.

### Attack
The attackers employed a supply chain attack that operated through multiple layers to attack specific users and to evade detection:

1. **Server-side filtering**: [The attack uses both server-side and client-side sampling](https://www.akamai.com/blog/security/2024-polyfill-supply-chain-attack-what-to-know). On the server side, [it examines request headers (such as the user-agent)](https://github.com/polyfillpolyfill/polyfill-service/issues/2873#issuecomment-2182491302) to target specific victims, serving a non-malicious version otherwise. Additionally, it selects targets based on the page referrer and the local information. 
2. **Client-side validation**: [When the request has a specific Referer and mobile UA, the server will return the JavaScript with malicious code, otherwise it returns the clean version](https://censys.com/blog/july-2-polyfill-io-supply-chain-attack-digging-into-the-web-of-compromised-domains) .
3. **Targeted redirection**: Victims meeting the criteria were subject to a malicious redirect.

Reports suggest that the attack specifically targeted the ```cdn.polyfill.io``` subdomain. However, while we did not use this subdomain specifically, the scope of compromise across subdomains remains unclear.

### Timeline
Our application deployments during the attack window were:

- **June 23, 2024** (git hash ```b82337b```) - Pre-attack deployment
- **June 25, 2024** - [Attack was disclosed](https://sansec.io/research/polyfill-supply-chain-attack)
- **June 27, 2024** - [Compromised domain was revoked](https://stackdiary.com/polyfill-io-gets-dealt-with-by-cloudflare-and-namecheap/)
- **June 29, 2024** (git hash ```a5a7de8```) - Post-takedown deployment  
- **June 30, 2024** (git hash ```9029318```) - Post-takedown deployment
- **June 10, 2025** - Removal of references in project repositories

While some sources report that [the attack started on June 25](https://www.invicti.com/blog/web-security/polyfill-supply-chain-attack-when-your-cdn-goes-evil/), this is not fully confirmed. If true, our systems and users may not have accessed the domain during the attack as do not use live CDNs for privacy and security reasons.

### Diagnosis
Given that public resources documenting the attack do not confirm a definitive date range of compromise, it is possible that we were making HTTP requests to the compromised domain and, thus, had potential exposure. Even so, our enhanced privacy and security practices provided multiple layers of protection:

1. **Privacy-First Architecture**: Our CI/CD systems do not expose user agent strings, user IP addresses, or referrer information when requesting resources to bundle. This likely prevented the server-side attack triggers from activating. This is used to avoid watering hole attacks (against our website) and user targeted attacks (against our visitors).
2. **Static Application Model**: Our fully client-side static application architecture reduces attack surface compared to server-side applications or those with local software install. This also means that users are not entering credentials into our webpages that could be sniffed by compromised code. Even so, user activity on the webpage is still considered private and could be monitored.
3. **Segregated Build Environment**: Our isolated build and deployment pipeline would have contained any potential CI/CD-targeted attacks impacting the software making the HTTP requests. In particular, the virtual machine with access to production credentials running the deployment step does not actually execute any project code beyond the upload itself. Therefore, communication with the compromised domain would not have impacted the VM with elevated access.

These practices should be enforced across other projects internally. Note that, while not a specific security measure, our **CI/CD Resource Caching** played a role in possibly making this a near-miss: We download and cache third-party resources during build time rather than serving them directly from CDNs, potentially limiting our exposure window.

## Response
Immediate actions were already taken but this offers further opportunities for future improvement.

### Immediate Actions
As part of the regular audit, the following actions were immediately taken:

- Removed all polyfill.io references.
- Verified historical deployments to confirm that the polyfill file was empty post-takedown in June 2024. This means there is a guarantee of no security impact after the community actions taken in June 2024.
- Updated cache busters to force application update.

We are also rotating production resources as a precautionary measure.

### Future Improvements
In addition to these measures, the following should be considered:

1. **Cryptographic verification**: Subresource Integrity (SRI) features aren’t relevant here as we do not access the CDN directly. However, as a similar measure, we should implement MD5 checksum validation for all external resources in CI/CD pipelines to detect unauthorized modifications, failing in CI / CD if the sources changed.
2. **CDN Provider Standards**: We have historically chosen the CDN recommended by the project but we may consider establishing organizational policy preferring specific CDN providers (Cloudflare, etc.) over project recommendations in the future.
3. **Enhanced Supply Chain Monitoring**: We rely heavily on automated scans but, in light of this attack, we should consider increasing manual third-party resource auditing frequency from annual to quarterly.

Finally, though not relevant to this specific attack, we should further restrict software and network access for production deployment systems to consider future supply chain attacks.

### Lessons Learned
At a high level, the following forces were at play in this incident:

1. **Privacy-first architecture provided security benefits** by shielding our users from targeted attack triggers.
2. **Supply chain attacks require more frequent monitoring**, specifically the current manual audit frequency should be increased.

Specifically, despite the CVE being published and the service being taken offline in June 2024, we didn't identify our exposure until June 2025 due to:

- Text-based repository searches failed to identify the dependency as the CDN subdomain specifically was not use.
- CDN-like dependencies aren't covered by automated vulnerability scanning (unlike PyPI packages).
- Manual auditing processes are currently infrequent.

Note that our currently employed website scanning would not likely detect this attack due to our CI/CD caching, requiring manual audit.

## Impact Assessment
There are limits to what we know for sure. From the public information available, it is not possible to know for certain what server-side logic was running on the compromised domain before it was revoked. Similarly, it is unclear when the attack was active though it appears actors such as the domain registrar moved quickly.

Based on the information made available by the security community, we believe the following:

 - It is very likely that the safety of service credentials (CI / CD, SSH keys, etc) was maintained during the attack.
 - It is very likely that any attack effects ceased shortly after the domain was revoked due to the timing of our deploys.
 - It is likely that, though not specifically listed as compromised, the subdomain we used was also subject to the attack.
 - It is likely that hiding the user's agent prevented the attack from triggering.
 - It is possible that the compromised service was not used while the attack was active.

Given the evidence linked from this retrospective, we are classifying this incident as a **near-miss** as we believe the attack was unlikely to have actually caused malicious code to reach user machines through our service.

## Notes
Thank you to Matt Fisher (@mfisher87) for feedback on this report. Though investigation and initial report drafting was mnaul, Anthropic Claude 4 was used to help with final report compilation, primarily for the purposes of formatting and proofreading. Its work was manually validated by the author.

An unrelated project for soil health also used polyfill io but similarly uses cached CDN payloads. It did not have deploys (was archived) prior to the sale of the project to the adversary. Therefore, no retrospective is needed.
