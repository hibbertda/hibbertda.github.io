---
layout: article
title:  "Invite external users with Logic Apps"
date:   2022-12-07 12:25:00 -0500
categories: Azure
tags: azure aad security logicApps invites
excerpt: "Default invitation email is ugly and bad. Can we do something better?"
header:
  theme: default
article_header:
  type: cover
  theme: default
  background_image:
    src: /assets/images/2022-12-07-aadinvite/finished-email.png
---
<meta property='og:title' content='Customize AAD Invite for external users with Azure Logic Apps'/>
<meta property='og:image' content='/assets/images/2022-12-07-aadinvite/finished-email.png'/>
<meta property='og:description' content='Default invitation email is ugly and bad. Can we do something better?'/>

Working on an application that needed to invite extneral users into our Azure Active Directory (AAD) tenant. Easy enough right? Mostly, maybe. Manually inviting users from the Azure portal is easy from the AAD user blade. Click '**New User**' and select the option to invite external users. Full in an email address and some basic config and it works, it does. My problem comes from the email the external user receives by default.

![Default AAD external invite email](/assets/images/2022-12-07-aadinvite/aadexinvitation.png){:.border.rounded}

Look at this thing. Include loads of detail in the text, which I’m sure most users will be familiar with (/s). No branding for the application, or even organization where the invite originated. It's UGLY. The warning while good and true might come off as a little scary without something more indicating where it came from.

Assume there was a better option out there I hit the internet and started my search.

## Microsoft Graph

Quickly that search lead me to the only real option to start down this road, the [Microsoft Graph API][msgraph-docs]. Which will give us all of the tools needed to take more control over the invite process. The [Invitation Manager][msgraph-invitation-manager] methods in Microsoft Graph enable creating new invitations with options not available through the Azure Portal.

### Invitation Manager

With a small JSON payload the Invitation Manager will take care of creating the invite. The cool part is this gives us the option of not sending the default invitation email to the external user. In that case the invite redemtion URL will be provided back in the response from AAD and we can do with it as we like.

#### Example payload

The reqest payload only requires basic information about the external user, though additional option are available from the method.

```json
{
  "inviteRedirectUrl": <SOME URL>,
  "invitedUserDisplayName": Joe P. Public,
  "invitedUserEmailAddress": Joe@public.something,
  "sendInvitationMessage": false
}
```

#### Example response

Graph will respond with all of the information regarding the invitation, and everything we need to peice together our own invitation. including:

- **inviteRedeemUrl**: URL needed to accept the invitation and complete onboarding.
- **inviteRedirectUrl**: URL the user will be redirected to once the onboarding is completed.

```json
HTTP/1.1 201 Created
Content-type: application/json

{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#invitations/$entity",
  "id": "7b92124c-9fa9-406f-8b8e-225df8376ba9",
  "inviteRedeemUrl": "https://invitations.microsoft.com/redeem/?tenant=04dcc6ab-388a-4559-b527-fbec656300ea&user=7b92124c-9fa9-406f-8b8e-225df8376ba9&ticket=VV9dmiExBsfRIVNFjb9ITj9VXAd07Ypv4gTg%2f8PiuJs%3d&lc=1033&ver=2.0",
  "invitedUserDisplayName": "Fabrikam Admin",
  "invitedUserEmailAddress": "admin@fabrikam.com",
  "resetRedemption": false,
  "sendInvitationMessage": false,
  "inviteRedirectUrl": "https://myapp.contoso.com",
  "status": "Completed",
  "invitedUser": { "id": "243b1de4-ad9f-421c-a933-d55305fb165d" }
}
```

## Logic App

Tying it all together with one of my favorite tools from the bag, an Azure Logic App. The Logic App can host all of the calls between services, generating a custom email, and sending everything out to the intended recipient.

### Trigger

![Logic App HTTP Trigger](/assets/images/2022-12-07-aadinvite/la-trigger.png){:.border.rounded}

### Create Invitation

![HTTP GraphAPI](/assets/images/2022-12-07-aadinvite/la-http-graph.png){:.border.rounded}

|---|---|
|Method|POST|
|Body|Parsed information from the incoming HTTP request to create the basic request to Microsoft Graph|
|Authentication|You can use several different authentication methods. In my case I wanted to make this easy to manage into the future so I used a system-assigned managed identity assigned to the Logic App|

#### Permissions

It's important to not forget the [required permissions][require-permissions] to use Microsoft Graph to create invitations.

|Permission Type|Permission|
|---|---|
|Application|User.Invite.All, User.ReadWrite.All, Directory.ReadWrite.All|

### Invite URL Variable

![InviteURL Variable](/assets/images/2022-12-07-aadinvite/la-url-variable.png)

Create a variable task to capture the '**inviteURL**' from the Microsoft Graph HTTP response. The variable will be used in the follow step to update the body of the email.

### Generate Invitation Email

![Variable to generate email](/assets/images/2022-12-07-aadinvite/la-genemail.png){:.border.rounded}

I use Logic App Variable task to define and modify the HTML for the email box. The HTML and previous variables are merged together and ready to use in the next step.

|---|---|
|Name|unique variable name, i used '**emailbody**'|
|Type|String|
|Value|HTML block for the email body. With includes for the **inviteURL** (@{variables(' inviteURL')})|

```html
<tr>
  <td class="bg_dark email-section" style="text-align:center;">
    <div class="heading-section heading-section-white">
      <h2>Welcome to TEAM</h2>
      <p>
              Welcome to the Team technical demo and showcase environment. Accept the invitation below to complete the onboarding process.
              you will automatically be directed to our catalog when finished.
          </p>
    </div>
  </td>
</tr>
<tr>
  <td class="bg_white email-section">
    <div class="heading-section" style="text-align: center; padding: 0 30px; font-size: 30px;">
              <a href="@{variables(' inviteURL')}" target="_blank" class="btn btn-primary h2">Accept invitation</a>                            
    </div>
        </td>
      </tr>                  
<tr>
  <td class="bg_white email-section">
    <div class="heading-section" style="text-align: center; padding: 0 30px;">
      <span class="subheading">Contact</span>
            <p>If there are any question please contact the team.</p>
            <a href="mailto:#" class="btn">Email TEAM</a>
    </div>
        </td>
      </tr>
```

This block is only the HTML that contains the inputs from the Logic App workflow. I used a started template from [Bootstrap Email](bootstrap-email) and added my bits where needed.
{: .info}

### Send Email

![Send email task](/assets/images/2022-12-07-aadinvite/la-sendemail.png){:.border.rounded}

The SendGrid Send Email services built-in to Azure includes Logic App tasks that can send email with out all of the extra headache of try and manage sending email on its own. There are [additional steps required][az-sendgrid-setup] to get your SendGrid account up and running before you can use the tasks. Follow the steps and create an API key, that will be used to authorize the task.

I put this together just before the release of the [Azure Communication Email Service][ACS-email]. There are logic app tasks now available for the new service, and they look a bit easier. Shouldn't be a problem to swap one for the other. I just haven't had the chance.
{: .info}

## Wrap it all Up

With all of those peices in place we get a customer email invetation sent off to the external user. Which can be customized to your organization, or application. 

![Finished custom invite email](/assets/images/2022-12-07-aadinvite/finished-email.png){:.border.rounded}

From here we have out own API for creating invitations. The next step is to create a front end that calls the Logic App to initial the process. That front end can be anything. 

[msgraph-docs]: https://learn.microsoft.com/en-us/graph/use-the-api
[msgraph-invitation-manager]: https://learn.microsoft.com/en-us/graph/api/resources/invitation?view=graph-rest-1.0
[require-permissions]: https://learn.microsoft.com/en-us/graph/api/invitation-post?view=graph-rest-1.0&tabs=http#permissions
[ACS-email]: https://learn.microsoft.com/en-us/azure/communication-services/concepts/email/email-overview
[az-sendgrid-setup]: https://docs.sendgrid.com/for-developers/partners/microsoft-azure-2021
[bootstrap-email]: https://bootstrapemail.com/



