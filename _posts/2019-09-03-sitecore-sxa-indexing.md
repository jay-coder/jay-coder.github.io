---
layout: post
title: Sitecore - SXA1.7 Indexing for CM
redirect_from:
  - /sitecore-sxa-indexing/
---

I recently encountered an issue that the CM site always retrieve data from production Solr (sitecore_sxa_web_index). Which means our marketing managers could not verify their changes on authoring environment until they publish contents to production. 

That's not ideal, why the out of box "Search Results" component not pull data from authoring Solr (sitecore_sxa_master_index)?


I tried to de-compile the SXA dlls but with no luck. Then I found [https://doc.sitecore.com/developers/sxa/17/sitecore-experience-accelerator/en/configure-sxa-indexing.html](https://doc.sitecore.com/developers/sxa/17/sitecore-experience-accelerator/en/configure-sxa-indexing.html) via Google.

After applying Site Grouping > [Site] > Indexing > Indexes to authoring site, it works, hooray!

<img src='{{ "/public/assets/img/sitecore_idx_auth.png" | relative_url }}' alt="Indexing Authoring" />

Then I applied the similar thing for production site.

<img src='{{ "/public/assets/img/sitecore_idx_prod.png" | relative_url }}' alt="Indexing Production" />

However, I found another "bug" in "JSON Results" component which tends to ignore the "Indexes" I set as above and pulls data from production still.

<img src='{{ "/public/assets/img/sitecore_json_results.png" | relative_url }}' alt="JSON Results" />

I raised this bug to Sitecore support. Hopefully they can supply a hotfix for it.