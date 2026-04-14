# CanisterWorm-Adjacent Outage

**Summary**: This retrospective documents how routine security hardening inadvertently caused an outage for new users and recently upgraded users within the Kigali Sim web-based IDE. There was no security or privacy risk to Kigali Sim users with the outage itself being the only impact.

<br>

## Incident Overview
**Projects**: [Kigali Sim](https://kigalisim.org/)

**Responders**: A Samuel Pottinger (@sampottinger)

**Report / CVE**: [CanisterWorm](https://socket.dev/blog/canisterworm-npm-publisher-compromise-deploys-backdoor-across-29-packages) (See also [LiteLLM vulnerability](https://www.endorlabs.com/learn/teampcp-isnt-done) and [JFrog Reporting](https://research.jfrog.com/post/canister-worm/))

**Attack Timeline**: CanisterWorm was reported around March 20. A proactive hardening change which led to outage was released on Friday April 10.

**Discovery Date**: Community reported about 8am April 14, 2026 (PT).

**Impact**: An outage lasting from April 10th, 2026 (PT) to April 14th, 2026 (PT) for web-based IDE users.

**Status**: Closed April 14, 2026 at 10am PT with preventative measures.

**Notes**: Users of Kigali Sim either through agentic AI or the CLI (the jar file) were not impacted.

<br>

## Technical Details
Kigali Sim's pre-existing security architecture prevented [CanisterWorm](https://socket.dev/blog/canisterworm-npm-publisher-compromise-deploys-backdoor-across-29-packages) from attacking the project or its users. However, further proactive hardening efforts inadvertently led to a service outage.

### Attack
[CanisterWorm](https://socket.dev/blog/canisterworm-npm-publisher-compromise-deploys-backdoor-across-29-packages) uses `npm` [post-install scripts](https://docs.npmjs.com/cli/v8/using-npm/scripts) to install malware as part of the dependency resolution process. In practice, this served to exfiltrate credentials and self-propagate to other packages. This followed earlier patterns established by [Shai-Hulud](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem).

### Mitigation
Kigali Sim previously had multiple steps to limit exposure to supply chain attacks. However, additional measures were added preemptively which led to an outage.

#### Prior mitigations
In observation of a changing threat landscape, the following were already in place prior to the attack:

1. *Credential segregation:* CI / CD segregates environments such that production credentials are only available on an ephemeral virtual machine with limited software present. Specifically, [this step](https://github.com/SchmidtDSE/kigali-sim/blob/main/.github/workflows/build.yaml#L537) only uploads a pre-built set of files produced by a different VM to production via SFTP. Therefore, `npm` post-install scripts could not access these credentials.
2. *Disabled post-install scripts:* Previous hardening moved to the [pnpm tool](https://pnpm.io/) from the [npm tool](https://github.com/npm/cli). This was done to avoid execution of post-install scripts required by [Shai-Hulud](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem) and [CanisterWorm](https://socket.dev/blog/canisterworm-npm-publisher-compromise-deploys-backdoor-across-29-packages).
3. *Version fixed dependencies:* We have preferred version-fixed pre-built dependencies so automatically moving to compatible upgraded versions was not allowed.
4. *Pre-fetch front-end resources:* All resources are pre-fetched and bundled as, for privacy reasons, we do not allow Kigali Sim to access CDNs from production.

#### New Mitigations
CanisterWorm did not pose a threat to Kigali Sim. However, in observing the changing threat landscape, we took additional proactive hardening steps:

 - Require front-end dependencies to go through Cloudflare with fixed versions.
 - All front-end dependencies are version pinned with [hash checking](https://github.com/SchmidtDSE/kigali-sim/blob/2a434224c3f2b7d2db1d1bd143e04ba7bdde8053/editor/support/validate_deps.py#L4).
 - All Github Actions are [hash-pinned](https://blog.rafaelgss.dev/why-you-should-pin-actions-by-commit-hash).
 - Simple actions were replaced with bash commands to reduce dependencies given [npm exposure in Github Actions](https://www.getsafety.com/blog-posts/shai-hulud-npm-attack).

Note that [ANTLR](https://www.antlr.org/) requires use of webpack so some limited dependencies still come through `npm` the service via `pnpm` the tool.

### Outage
The default target for our [Chart.js](https://www.chartjs.org/) dependency uses a different import pattern ([UMD vs ESM](https://dev.to/iggredible/what-the-heck-are-cjs-amd-umd-and-esm-ikm)). We inadvertently moved to ESM when switching sources (jsdelivr to cloudflare) while the full application expected UMD. This was caused by the two sources not having the same file naming scheme. Regardless, this led to visualizations on the front-end to break.

### Timeline

 - 2025-09-18: [Completed prior mitigations](https://github.com/SchmidtDSE/kigali-sim/pull/556)
 - 2026-03-20: CanisterWorm reported
 - 2026-03-21: Confirmed no Kigali Sim exposure to CanisterWorm
 - 2026-04-10: Pre-emptive hardening and outage
 - 2026-04-14: Resolution

### Diagnosis
This took some time to identify and resolve:

 - The mechanics of running Kigali Sim itself both in web browser and outside were still functional. Therefore, both automated Java and JS tests still passed. However, the failure in the chart was not captured in CI / CD.
 - We do not allow use of [Sentry](https://sentry.io/welcome/) or similar for automated error reporting for privacy reasons. Instead, users are prompted to report the issue to us using an automated but opt-in private process.
 - Users which previously visited the app may have still used a PWA / cached version with lazy upgrade.

Altogether, this caused a delay in reporting the issue to us. That said, the live JS error encountered in browser (ESM imports not allowed) when visited without a cached version (like through incognito) identified Chart.js as the broken dependency.

<br>

## Response
We were able to switch to the UMD bundle to restore functionality but, due to PWA caching and the location of the error, some users may need to hard refresh.

### Immediate Actions
We were able to use a compatible hash-pinned version of Chart.js through Cloudflare but with the UMD option instead. [See relevant PR #792](https://github.com/SchmidtDSE/kigali-sim/pull/792/changes#diff-1056f211264dd2b48082ea1b6f7d6c2efef9750c0779d38f7a92c0ad2a5c5453R53).

### Further Improvements
Although CI / CD tests the front-end through a live browser, there was not a step to test the IDE as a whole after making the final bundle. Therefore, an additional "smoke" test loads the completed bundled web application in a live browser in CI / CD and looks for errors in initialization. This would have caught this issue and prevented production deploy through automation.

<br>

## Reflection
Some things went well in our security posture that prevented a larger issue but improvements are needed for reliability.

### Went well
All of the steps taken offer security benefit and an outage is preferable to a compromised release. Additionally, integration tests did exist in the form of [WASM](https://github.com/SchmidtDSE/kigali-sim?tab=readme-ov-file#public-hosted-browser-based-app) execution of example scripts as well as checks on the final Java package ([Docker](https://github.com/SchmidtDSE/kigali-sim?tab=readme-ov-file#docker-cli) image-based validation prior to release to [Maven](https://github.com/SchmidtDSE/kigali-sim/packages/2723826)). Therefore, the core package was always reliable. Finally, prior security hardening steps prevented potential exposure for Kigali Sim in this and other recent incidents.

### Improvements
Beneficial changes in CI / CD carry particular risk as there isn't a higher order mechanism to test their validity. This fell through in the front-end. **We regret deeply that the same care extended to the core package was not given to the web-based IDE.** This component is an important part of our user experience in Kigali Sim. This as well as other projects which offer a similar [programming portal](https://maggieappleton.com/programming-portals) should ensure that full "smoke tests" offer some kind of "integration test" functionality through a live web browser on the final bundle. We sincerely apologize for this oversight and, to our user community, we regret the outage.

### Looking forward
We have already taken steps to improve our reliability practices and we will continue to extend the surface area of these checks into the future. We recognize that this expanded proactive validation prior to release is essential given that, for privacy reasons, we do not allow automated bug reporting from the live application.

<br>

## Impact Assessment
No security or privacy risk. However, 4 days (2 workdays, 2 weekend days) outage for new users in the web-based IDE. This excluded those using a pre-loaded PWA / cached copy, local jar, or agentic AI flow, although these other pathways delayed detection.

<br>

## Notes
This is posted in the Kigali Sim repository as [#791](https://github.com/SchmidtDSE/kigali-sim/issues/791) and archived in the [DSE retrospectives repository](https://github.com/SchmidtDSE/retrospectives). This was reviewed by [@GondekNP](https://github.com/GondekNP) and Kigali Sim thanks him for his contribution.

Due to our privacy-first architecture, direct proactive notification of affected users was not possible. The resolution was communicated via GitHub issue [#796](https://github.com/SchmidtDSE/kigali-sim/issues/796).

[Claude](https://claude.ai/) proofread this document and offered minor refinements but it was originally drafted manually.
