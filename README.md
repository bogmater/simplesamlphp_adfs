# Configuring SimpleSAMLphp as SP to work with ADFS as IDP

## SimpleSAMLphp configuration

1) Download ADFS Metadata

- Find the ADFS Metadata url - open Tools -> AD FS Management, on the left menu go to Service -> Endpoints, scroll down to Metadata, verify that Metadata with type "Federation Metadata" is enabled and copy the URL Path. Default value should be something like /FederationMetadata/2007-06/FederationMetadata.xml.
- visit your configured AD server and add the Metadata URL you just copied (e.g. http(s)://your.ad.com/FederationMetadata/2007-06/FederationMetadata.xml) and you will download a .xml file with the ADFS metadata.

2) Convert XML Metadata to php

- visit http(s)://your.simplesaml.sp/simplesaml/admin/metadata-converter.php
- copy the ADFS Metadata from the .xml file and input it to the "XML Metadata" textbox - click Parse and copy the php code the metadata converter outputs

3) Add the IDP metadata to the SimpleSAMLphp SP
- go to your SimpleSAMLphp SP installation folder (e.g. /path/to/simplesaml) and append the php code you just copied to /path/to/simplesaml/metadata/saml20-idp-remote.php file.

This process should ensure that your SimpleSAMLphp SP knows about the ADFS IDP.

The next step is letting the ADFS know about the SP that will be connecting to it as the IDP.

## ADFS configuration

1) Add relaying party trust to ADFS (basically a copy of [this guide](https://technet.microsoft.com/en-us/library/dd807132.aspx))

- open Tools -> AD FS Management
- on the left side click Trust Relationships -> Relying Party Trusts
- on the right side click Add Relying Party Trust (a wizard will open)
- click Start
- **NOTE:** importing data from relying party online or on a local network should work - but I couldn't get it to work. I researched the issue a little bit and it looks like it's some kind of a problem with HTTP 1.1 chunking, but didn't go into too much detail because the alternative (import data about relying party from a file) works just as well.
- downlad your SimpleSAMLphp SP metadata from http(s)://your.simplesaml.sp/simplesaml/module.php/saml/sp/metadata.php/{authsource_name} - in the case of the test environment this url is https://5qfgdriltoqi.schoolzilla.com/simplesaml/module.php/saml/sp/metadata.php/adfs
- choose Import data about the relying party from a file and choose the file you just downloaded
- continue with your specific settings (display name, multi factor, issuance authorization rules)

2) Edit Claim Rules for the relaying party you just added to expose desired attributes

- click Edit claim rules when you select the relaying party trust you just created
- make sure you are in the "Issuance Transform Rules" tab
- click the "Add Rule..." button and the Add Transform Claim Rule Wizard will open
- choose Send LDAP Attributes as Claims as the Claim rule template (dropdown default)
- Give a name to the Claim rule (LDAP attributes or something), select Active Directory as attribute store and add the desired attributes (choose LDAP Attribute and give it a custom name) you want to be passed to the SimpleSAMLphp SP, and Finish.

3) Edit Claim rules for the relaying party you just added to issue a NameID

- click Edit claim rules when you select the relaying party trust you just created
- make sure you are in the "Issuance Transform Rules" tab
- click the "Add Rule..." button and the Add Transform Claim Rule Wizard will open
- choose "Transform an Incoming Claim" as the Claim rule template (from the dropdown)
- Give a name to the claim rule ( *something* to nameID usually), choose the desired attribute you want to be sent as the NameID from the Incoming claim type dropdown, choose "Name ID" as the Outgoing claim type, and choose "Transient Identifier" as the Outgoing name ID format, and Finish.

This finishes up the ADFS configuration - but some additional notes are required.

**IMPORTANT**: make sure that the attribute you transform to the Name ID is something that is 
- unique (self explanatory)
- present in all users

If a user does not have the attribute (for instance - your NameID transform takes the E-mail attribute and transforms it to NameID, and the user does not have the E-mail attribute defined) - the login will fail with an "Invalid NameID Policy " Exception on the SimpleSAMLphp SP side and it's going to be hard to track down, because it's actually not a NameID Policy issue, but rather an error on the AD side that happens because there is no attribute to transform to the NameID. I've spent an entire day on debugging this thinking it's a SimpleSAMLphp issue, but it turned out to be a simple missing attribute problem on the test user I was logging in with.

OpenLDAP on Linux has an attribute 'entryUUID' which is a unique identifier 

This is all you should know for how to configure SimpleSAMLphp and ADFS to work together properly.

## SimpleSAMLphp authsource

The last part that needs explaining is how to add the ADFS authsource to SimpleSAMLphp.

You need to edit the /path/to/simplesaml/config/authsources.php file.

Append the following code:

```php
    'adfs' => [
                'saml:SP',
                'entityID' => NULL,
                'NameIDPolicy' => 'urn:oasis:names:tc:SAML:2.0:nameid-format:transient',
                'idp' => 'http://adfspoc.schoolzilla.com/adfs/services/trust'
        ]
```

Also, please take note that this process may be customized in many ways and your MMV with this guide and you will probably have to change multiple steps if you change one (for instance, if you want to use persistent NameID format in the ADFS claim transform to persistet, you will have to use a different value for the 'NameIDPolicy' entry in the SimpleSAMLphp authsorce, something like 'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent' should do the job (untested).), but hopefully it's a good base guide for getting SimpleSAMLphp and ADFS to play well together.
