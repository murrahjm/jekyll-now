---
layout: post
title: Displaying JSON in a Markdown Table
featured-img: Powershell
comments: true
categories: [PowerShell]
---
This will be a quick article, and hopefully useful to some folks.
I came across this while trying to find a way to self-document some data from a REST API.
In my specific case, I'm working with the awx credential type data, but this method should work for any list data that you want to display cleanly.
So let's just dive right in.
Here is a sample of the data I retrieve from the REST API endpoint.

```json
[
        {
            "id": 16,
            "type": "credential_type",
            "url": "/api/v2/credential_types/16/",
            "related": {
                "credentials": "/api/v2/credential_types/16/credentials/",
                "activity_stream": "/api/v2/credential_types/16/activity_stream/"
            },
            "summary_fields": {
                "user_capabilities": {
                    "edit": false,
                    "delete": false
                }
            },
            "created": "2020-01-29T22:05:41.098632Z",
            "modified": "2020-01-29T22:06:07.813040Z",
            "name": "Ansible Tower",
            "description": "",
            "kind": "cloud",
            "namespace": "tower",
            "managed_by_tower": true,
            "inputs": {
                "fields": [
                    {
                        "id": "host",
                        "label": "Ansible Tower Hostname",
                        "type": "string",
                        "help_text": "The Ansible Tower base URL to authenticate with."
                    },
                    {
                        "id": "username",
                        "label": "Username",
                        "type": "string"
                    },
                    {
                        "id": "password",
                        "label": "Password",
                        "type": "string",
                        "secret": true
                    },
                    {
                        "id": "verify_ssl",
                        "label": "Verify SSL",
                        "type": "boolean",
                        "secret": false
                    }
                ],
                "required": [
                    "host",
                    "username",
                    "password"
                ]
            },
            "injectors": {
                "env": {
                    "TOWER_HOST": "{{host}}",
                    "TOWER_USERNAME": "{{username}}",
                    "TOWER_PASSWORD": "{{password}}",
                    "TOWER_VERIFY_SSL": "{{verify_ssl}}"
                }
            }
        },
```

This is one element in a long list of similar objects.
Great info, but not much for the eyes.
In my case, I want to display only the `name`, `inputs`, and `injectors` fields.
Because I have a long list of items, a table format with three columns would be ideal.
Also, because the `inputs` and `injectors` data contains complex sub elements, I want to display those in a way that preserves the object structure.
YAML should work great for this purpose.
So ultimately, this is what i would like to see:

<table>
<colgroup><col/><col/><col/></colgroup>
<tr><th>name</th><th>inputs</th><th>injectors</th></tr>
<tr><td>Ansible Tower</td><td>

```
- id: host
  label: Ansible Tower Hostname
  type: string
  help_text: The Ansible Tower base URL to authenticate with.
- id: username
  label: Username
  type: string
- id: password
  label: Password
  type: string
  secret: true
- id: verify_ssl
  label: Verify SSL
  type: boolean
  secret: false
```

</td><td>

```
env:
  TOWER_HOST: '{{host}}'
  TOWER_USERNAME: '{{username}}'
  TOWER_PASSWORD: '{{password}}'
  TOWER_VERIFY_SSL: '{{verify_ssl}}'
```

</td></tr>
</table>

Nice clean output, everything organized and readable markdown.
Well technically it's a combination of markdown, yaml and html.
So how do we get there?
Well here's a relatively short script

```powershell
# make api call to get the data into the $data variable
# convert the data from json to PowerShell objects for easy manipulation
$cred_types = $data | convertfrom-json
# loop through the objects in the list to pick out the properties we care about
$output = foreach ($type in $cred_types){
    $props =  [ordered]@{
        name=$type.name
        # the magic happens here, a little yaml mixed with some markdown codeblock markers
        inputs="`r`n`r`n" + '```' + "`r`n" + $($type.inputs.fields | convertto-yaml) + '```' + "`r`n`r`n"
        injectors="`r`n`r`n" + '```' + "`r`n" + $($type.injectors | convertto-yaml) + '```' + "`r`n`r`n"
    }
    new-object psobject -property $props
}
# convert the resulting powershell object to html, and fix up the one URI encoding error
$html = ($output | convertto-html) -replace "&#39;","'"
$html | out-file .\Ansible\Ansible-Tower\Tower-Credential-Types-Data.md -Force
```

As mentioned, there are several things going on here.
In order to display properly, a markdown table didn't suffice, I had to resort to an HTML table to get everything to line up right.
The next trick is the display of our conplex objects.
converting to yaml works great for getting the data in that readable format, but preserving the spacing was difficult, as everything kept trying to clean it up.
Ultimately, a markdown code block served that purpose well, as it isolated the yaml from any parsing or formatting attempt, while itself fitting inside of a table cell.
Also not the multiple new-lines, as markdown requires an empty line before and after any code blocks, for proper display.

There's not much more to say about it, it's pretty straightforward, if not a little convoluted.
I originally created it for a single project (blog post on that incoming), but in theory it should work for just about any API data that you might want to display.
If you use it for something else, I'd love to hear about it.
Post in the comments for find me on twitter and let me know!
