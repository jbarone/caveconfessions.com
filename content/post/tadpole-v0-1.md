+++
date = "2014-07-25T19:09:00"
draft = "false"
title = "TADpole v0.1"
slug = "tadpole-v0-1"

+++

Today is my birthday. so what did I do with this day? I decided to finally get around to releasing TADpole. TADpole is the tool I created as part of my master's thesis "[Automated Timeline Anomaly Detection](http://scholarworks.uno.edu/td/1609/ "Automated Timeline Anomaly Detection")." To be honest I've been hesitant to release this. I am a perfectionist and the code was written to prove out the research. But, I'm finally getting over my ego, and I present to you TADpole.

#### What is TADpole ####

TADpole is tool designed to aide forensic investigators during the initial triage of a system. The purpose of the tool is to determine if the system clock of the machine had been purposefully manipulated. An example of manipulation in this case would be someone changing the clock back two weeks to alter accounting information, then resetting the clock to current time. Currently TADpole only works for Windows systems.

#### How does it work ####

TADpole searches each file on a forensic disk image looking for Windows event log files. It then looks for entries in the log files that are anomalous. Once these anomalies are found, they are then correlated across multiple log files to increase the certainty that this anomaly is a true anomaly and not a mistake in the log file.

#### Where can you get it ####

The source code is available on my GitHub page [https://github.com/jbarone/tadpole](https://github.com/jbarone/tadpole "https://github.com/jbarone/tadpole").