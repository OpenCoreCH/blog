+++
title = "Using SavaPage as a Frontend for uniFLOW"
date = "2022-01-12"
author = "Roman Böhringer"
cover = ""
tags = ["sre"]
keywords = ["canon", "uniflow", "print", "savapage", "byod", "printportal"]
description = "While the integration is relatively easy, ensuring that Follow-Me-Printing works correctly can be tricky. This article describes the necessary configuration steps."
showFullContent = false
+++

A [uniFLOW](https://www.uniflowonline.com) Queue (that is available over SMB) can be integrated quite easily into [SavaPage](https://www.savapage.org/) with the CUPS SMB backend. To add it, simply run the following command on the SavaPage server:
```sh
lpadmin -p uniflow_queue -v smb://serviceaccount:servicepassword@domain/uniflow_server/uniflow_queue -P path/to/driver.ppd
```
While this works, all print jobs will arrive at the uniFLOW server with the used `serviceaccount`.
SavaPage sends the username of the user that submitted the job in the `For` attribute of the transmitted PostScript file.
However, uniFlOW considers in its default configuration only the user that submitted the print job to the SMB queue and ignores this attribute.
Follow-Me-Printing will therefore not work, as all print jobs are assigned to the account `serviceaccount`.

To resolve the problem, you have to create a custom print workflow (ideally on a dedicated queue for SavaPage) where the `For` attribute is considered. Such a workflow could look like this:

![Example uniFLOW Workflow](/images/uniflow_for_comment_workflow.png)

The crucial step here is the second, "Identify user by For comment", which ensures that the attribute is used for the identification of the user. The other steps may look different, depending on your setup.
When you add a queue with such a workflow to SavaPage, the jobs are assigned to the correct user.

If you need help with the integration or want to use SavaPage in a different environment, OpenCore GmbH can help you to do so as a [Deployment Partner](https://wiki.savapage.org/doku.php?id=partner_list#opencore_gmbh).
