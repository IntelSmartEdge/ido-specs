```text
SPDX-License-Identifier: Apache-2.0
Copyright (c) 2020 Intel Corporation
```

# Support for overriding the startup command of a container in OpenNESS On-Prem mode

## Overview

This feature enables you to override the startup command for a container, thus removing the need to rebuild it just to make this change.
It also allows you to create multiple containers using the same image but with each container using a different startup command.

## Usage
To take advantage of this feature, all you have to do is add a new 'EPA Feature Key' (on the application details page) called 'cmd',
with the value of the command you want to run instead of the default. OpenNESS will pass that information down to Docker, and assuming all goes well (for example your command is correct / the path is valid), next time you start this container your command will be run.

For more details on the application onboarding (including other fields to set), please refer to 
[Application Onboarding Document](https://github.com/otcshare/x-specs/blob/master/doc/applications-onboard/on-premises-applications-onboarding.md) 
