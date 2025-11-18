```
---
Title: Troubleshooting the Hub Installation
Author: karshi.hasanov@utoronto.ca
Date: Nov. 17, 2025
Description: I had a trouble of installing the AAP Bundle 2.4-13.4 for Hub.
Last Modified Date: 
---
```
***

## TASK [ansible.automation_platform_installer.certificate_authority : set_fact] 
### ERROR: FAILED! => ...tasks/sign_service.yml': line 32, column 7
## Hacking the `sign_service.yml`
### Step 1: Cleaning up the current installation
```bash
# On hub.karshi.ca:
sudo rm -rf /etc/pulp/certs/*
sudo rm -rf /etc/ansible-automation-platform/*
sudo rm -rf /var/lib/awx/pki/

# On your installer host, clean any cached facts:
rm -rf ~/.ansible/cp/
rm -rf ~/.ansible/tmp/
```
## Step 2: Modifying the 'sign_service.yml` to bypass the check
Login to "controller.karshi.ca" and go to the AAP Bundle location
```bash
cd ~/aap/*latest
```
Edit the file:
*collections/ansible_collections/ansible/automation_platform_installer/roles/certificate_authority/tasks/sign_service.yml*.  
Find the failing **set_fact task** around line 32 and modify the condition to handle *None*:

```yaml
# Change FROM:
when: aap_service_san_records_internal | union(aap_service_san_records) | difference(certificate_check.subject_alt_name) | length > 0

# Change TO:
when: certificate_check.subject_alt_name is none or aap_service_san_records_internal | union(aap_service_san_records) | difference(certificate_check.subject_alt_name) | length > 0
```

```bash
./setup.sh -i invenyory_hub
```

## Great debugging!
That workaround confirms it's a legitimate bug in the AAP 2.4 installer—the code doesn't handle the case where certificate_check.subject_alt_name is None (which happens when no cert exists yet or when the OpenSSL parser can't find SANs).
## What This Reveals
This is a classic "happy path" testing gap: the condition works fine when a certificate exists but fails on fresh installations or after certificate cleanup. The installer should initialize certificate_check.subject_alt_name to an empty list [], not None.
## Professional Path Forward

1. Open a Red Hat Support Case:
   - Reference KB article 7079025
   - Mention you hit this on AAP 2.4-13.4 bundle
   - Provide the exact error and your fix
   - This pushes them to release an official patch
2. Document Your Change:
	```bash
   	# In your installer directory
	git diff collections/.../sign_service.yml > aap-2.4-cert-bug-workaround.patch
   	```
   	Keep this patch for future runs or team documentation.

3. Revert After Official Fix:
Once Red Hat releases a corrected collection, remove your manual edit and upgrade:
```bash
./setup.sh -e automation_platform_collection_upgrade=true
```
## Should I Be Frustrated?
Yes, but context helps: AAP 2.4 introduced a significantly refactored certificate management system for the new gateway architecture. These edge cases often slip through despite Red Hat's QA because they can't test every possible starting state (fresh install vs. upgrade vs. partial cleanup).

**The real shame** is that this bug has existed since at least 2.4-7 (per KB 7079025) and wasn't fixed in 2.4-13.4, suggesting it's not prioritized.






