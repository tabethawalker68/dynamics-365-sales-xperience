![Kentico.Xperience.Libraries 13.0.0](https://img.shields.io/badge/Kentico.Xperience.Libraries-v13.0.0-orange)

# Xperience Dynamics 365 Sales integration

This integration utilizes two scheduled tasks to synchronize Xperience contacts and activities to a Dynamics 365 tenant. It also contains custom Marketing automation actions to log four out-of-the-box Dynamics 365 activity types.

## Set up the environment

### Import the custom module

1. Download the latest export package from the [/CMSSiteUtils/Export](/CMSSiteUtils/Export) folder.
1. In the Xperience adminstration, open the __Sites__ application.
1. [Import](https://docs.xperience.io/deploying-websites/exporting-and-importing-sites/importing-a-site-or-objects) the downloaded package with the __Import files__ and __Import code files__ [settings](https://docs.xperience.io/deploying-websites/exporting-and-importing-sites/importing-a-site-or-objects#Importingasiteorobjects-Import-Objectselectionsettings) enabled.
1. Perform the [necessary steps](https://docs.xperience.io/deploying-websites/exporting-and-importing-sites/importing-a-site-or-objects#Importingasiteorobjects-Importingpackageswithfiles) to include the following imported folder in your project:
   - `/CMSModules/Kentico.Xperience.Dynamics365.Sales`

### Enable the integration

1. In the Xperience administration, open the __Settings__ application and navigate to __Integration → Dynamics 365__.
1. Add your Dynamics 365 Sales tenant to the __Dynamics URL__ setting. For example, if you access the application at https://mycompany.crm4.dynamics.com/main.aspx, enter _"https<area>://mycompany.crm4.dynamics.com."_ If you don't have  an instance, you can set up a new one using [Microsoft's documentation](https://docs.microsoft.com/en-us/dynamics365/sales/set-up-dynamics-365-sales).
1. To authenticate requests to Dynamics 365, [register an application](https://docs.microsoft.com/en-us/previous-versions/dynamicscrm-2016/developers-guide/mt622431(v=crm.8)#register-an-application-with-microsoft-azure) in Microsoft Azure Active Directory's __App registrations__ tab.
1. After registration, select the application registration, note the following in the Azure Portal, and set the corresponding Xperience setting:
   - Application (client) ID
   - Directory (tenant) ID
1. Click the __Certificates & secrets__ tab, navigate to the __Client secrets__ tab, and create a new secret for the integration. Add this secret to the __Secret__ setting in Xperience.

### Enable contact synchronization

Contact synchronization is performed by the scheduled task "Dynamics 365 contact synchronization" which is set to run every hour by default. However, you must first enable the synchronization by going to the __Settings__ application, navigating to the __Integration → Dynamics 365__ section, and checking the __Enabled__ box under the "Contact synchronization" category.

The scheduled task will only synchronize contacts that have reached a minimum total score. Ensure that you have the [contact scoring](https://docs.xperience.io/on-line-marketing-features/managing-your-on-line-marketing-features/contact-management/scoring-contacts) functionality configured properly, then set the minimum score in the __Minimum score__ setting. The score must be greater than zero.

This integration uses two custom contact fields to store information about contact synchronization, which must be created manually:

1. In the Xperience administration, open the __Modules__ application.
1. Edit the Contact management module → __Classes__ tab __→ Contact__ class __→ Fields__ tab.
1. Add two __New fields__ with the following configuration:

   - __Field name__: ContactDynamics365RelatedID
      - __Data type__: Unique identifier (GUID)
      - __Display field in the editing form__: No
   - __Field name__: ContactDynamics365SynchronizedOn
      - __Data type__: Date and time
      - __Precision__: 7
      - __Display field in the editing form__: No

Finally, you must map the Xperience contact fields to the desired Dynamics 365 fields. Click the _"Edit"_ button next to the __Field mapping__ setting to open a new dialog window. This window displays the available Xperience contact field (including any custom fields added by your developers), and a drop-down list containing the Dynamics 365 fields. For each Xperience field, select the Dynamics 365 that it will be mapped to. If you leave a drop-down on _"(not mapped)"_, that Xperience field will not be synchronized to Dynamics 365.

### Enable activity synchronization

Activity synchronization is performed by the scheduled task "Dynamics 365 activity synchronization" which is set to run every hour by default. However, you must first enable the synchronization by going to the __Settings__ application, navigating to the __Integration → Dynamics 365__ section, and checking the __Enabled__ box under the "Activity synchronization" category. Activity synchronization is not required for contact synchronization, but if you want to synchronize activities, you _must_ enable contact synchronization.

After enabling the synchronization, you can optionally choose a Dynamics 365 user or team in the __Default owner__ drop-down. All synchronized activities will be assigned to this user or team when they are created. If you leave this setting on _"(not set)"_ the activities will be unassigned.

## How the synchronization works

After [setting up the environment](#set-up-the-environment), you will find two new scheduled tasks in the __Scheduled tasks__ application of your administration. These tasks are enabled and set to run every hour by default, though you can disable them or adjust the interval as needed:

   - Dynamics 365 contact synchronization
   - Dynamics 365 activity synchronization

### Contact synchronization

 When the contact synchronization task runs, it will collect all Xperience contacts that have a score equal to or greater than the __Minimum score__ setting. The full contact data is sent to Dynamics 365 according to how you've [mapped the fields](#enable-contact-synchronization), and the ID of the Dynamics 365 contact that was created is stored in a custom Xperience field named "ContactDynamics365RelatedID." If activity synchronization is enabled, the contact's activities are also synchronized during creation of the contact.

Contacts that were _already_ synchronized in the past (e.g. the "ContactDynamics365RelatedID" fields contains a Dynamics 365 contact ID) are synchronized again during the task execution. The Xperience contact data is compared to the Dynamics 365 contact data to check whether the information was updated since the last synchronization. If no mapped fields have changed, synchronization is skipped for that contact. If some fields have changed, a partial update is made to the Dynamics 365 contact to avoid unwanted triggering of workflows (e.g. if you have a process that runs whenever a contact's email address is updated).

### Activity synchronization

This scheduled task loads all contacts that have been synchronized to Dynamics 365 and synchronizes their activities which were logged since the task's __Last run__ time. There is one exception- since activities are synchronized immediately when a new contact is created in Dynamics 365, this task will use the "ContactDynamics365SynchronizedOn" custom field to load only the activities logged after the synchronization time.

In order for an activity to be synchronized, the [activity type](https://docs.xperience.io/on-line-marketing-features/configuring-and-customizing-your-on-line-marketing-features/configuring-activities/adding-custom-activity-types) __Code name__ in Xperience must match the name of the entity in Dynamics 365. For example, Dynamics 365 contains the ["phonecall" entity](https://docs.microsoft.com/en-us/dynamics365/customer-engagement/web-api/phonecall?view=dynamics-ce-odata-9) out-of-the-box, which you can see by opening the __Power apps__ application and navigating to __Tables → Data__:

![Dynamics 365 entities](/Assets/entities.png)

To synchronize this activity from Xperience to Dynamics 365, this integration includes a custom activity type whose __Code name__ matches the Dynamics 365 entity name:

![Phone call activity](/Assets/phonecall.png)

You can customize this behavior by developing your own implementation of [`IDynamicsEntityMapper`](/CMSModules/Kentico.Xperience.Dynamics365.Sales/Services/IDynamics365EntityMapper.cs). The `MapActivityType` method is called while synchronizing activities to determine the name of the Dynamics 365 entity to create, using the passed `activityType` parameter. For example, if you have a custom entity in Dynamics 365 named "pageview" and you want to synchronize the Xperience activity "pagevisit," your implementation would look something like this:

```cs
public string MapActivityType(string activityType)
{
   if (activityType.Equals(PredefinedActivityType.PAGE_VISIT, StringComparison.OrdinalIgnoreCase))
   {
         return "pageview";
   }

   return activityType;
}
```

## Logging Dynamics 365 activities

This integration includes two custom Xperience [activity types](https://docs.xperience.io/on-line-marketing-features/configuring-and-customizing-your-on-line-marketing-features/configuring-activities/adding-custom-activity-types) that can be synchronized to Dynamics 365. You can log and synchronize these activities to Dynamics 365 to track how your marketers interacted with your Xperience contacts:

- Phone call
- Email

You can log these activities [via code](https://docs.xperience.io/on-line-marketing-features/configuring-and-customizing-your-on-line-marketing-features/configuring-activities/logging-custom-activities-through-the-api) as usual, but this integration also includes two custom __Marketing automation__ actions to help you log them:

- Log Dynamics 365 phone call
- Log Dynamics 365 email

You can add these actions to your [__Marketing automation__ processes](https://docs.xperience.io/on-line-marketing-features/managing-your-on-line-marketing-features/marketing-automation/working-with-marketing-automation-processes) to log Xperience activities, which will be synchronized the next time the "Dynamics 365 activity synchronization" scheduled task runs. To log the __Phone call__ activity, place it after an __Approve progress__ step:

![Phone call automation process](/Assets/phonecallprocess.png)

In this example, the contact submits a form on your site requesting more information about a product. The process triggers and waits in the "Call contact" step, where one of your marketers calls the contact to discuss your offerings. After the call, the marketer [approves the progress](https://docs.xperience.io/on-line-marketing-features/managing-your-on-line-marketing-features/marketing-automation/managing-the-flow-of-contacts-in-automation-processes#Managingtheflowofcontactsinautomationprocesses-Movingcontactsbetweenstepsintheprocess) and can also leave a comment about the call using __Comment and move to step__. The activity will then be logged including the information found in [`Dynamics365PhoneCallModel`](/CMSModules/Kentico.Xperience.Dynamics365.Sales/Models/Activities/Dynamics365PhoneCallModel.cs). If the marketer's email address matches an email address in Dynamics 365, that user will be linked to the Dynamics 365 activity.

To log the __Email__ activity, place it after a __Send transactional email__ or __Send marketing email__ step:

![Email automation process](/Assets/emailprocess.png)

When the activity is logged, it will contain the information found in [`Dynamics365EmailModel`](/CMSModules/Kentico.Xperience.Dynamics365.Sales/Models/Activities/Dynamics365EmailModel.cs), such as the email's subject and body. If the email's "From" address matches a user in Dynamics 365, that user will be linked to the Dynamics 365 activity.

> __Note__: If the "Log Dynamics 365 email" step is placed after a "Send marketing email" step, the body of the email will not be included in the Dynamics 365 activity.

There are two other custom automation actions which do not log Xperience activities, but will create Dynamics 365 entities _when they execute_:

- Create Dynamics 365 task
- Create Dynamics 365 appointment

With the __Create Dynamics 365 task__ action, you can create a task for your Dynamics 365 users to complete. For example, if you want to send flowers to everyone who submits a form on your site, the process could look like this:

![Task automation process](/Assets/taskprocess.png)

The task will be assigned to the user or team specified by the __Default owner__ setting, or unassigned if not set. You can also create an appointment in Dynamics 365 using the __Create Dynamics 365 appointment__ action. For example, if a partner submits a form indicating they'd like to have a meeting to discuss a new opportunity, the process might look like this:

![Appointment automation process](/Assets/appointmentprocess.png)

The appointment can contain required and optional attendees that you choose from your Dynamics 365 users. The appointment will always include the contact that is currently in the automation process.