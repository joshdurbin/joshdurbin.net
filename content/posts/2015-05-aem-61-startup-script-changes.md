+++
date = "2015-05-15"
title = "A quick note regarding one of AEM 6.1′s start script changes..."
tags = [ "aem" ]
+++

There is a persistence mode section of the AEM 6.1 (load23a) script that essentially concatenates the runmode set on line 24 with the the addition of `crx3,crx3tar` or `crx3,crx3mongo`. By default, the TarMK option is defined.

This is important because the underlying persistence systems that read the runmode seem to prefer tar in the case that both tar and mongo are defined.

See lines:

    59 # ------------------------------------------------------------------------------
    60 # persistence mode
    61 # ------------------------------------------------------------------------------
    62 # the persistence mode can not be switched for an existing repository
    63 #CQ_RUNMODE="${CQ_RUNMODE},crx3,crx3tar"
    64 CQ_RUNMODE="${CQ_RUNMODE},crx3,crx3mongo"

