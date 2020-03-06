# Spoke

Spoke is an open source text-distribution tool for organizations to mobilize supporters and members into action. Spoke allows you to upload phone numbers, customize scripts and assign volunteers to communicate with supporters while allowing organizations to manage the process.

Spoke was created by Saikat Chakrabarti and Sheena Pakanati, and is now maintained by MoveOn.

This is the fork of Spoke developed by Elizabeth Warren's 2020 presidential campaign.
It was built in the last month of the campaign and was used by thousands of volunteers to send millions of texts a day
in the run-up to Super Tuesday. It is based off of the [staging_20191102](https://github.com/WorkingFamilies/Spoke/tree/staging_20191102)
branch of the Working Families Party's fork of MoveOn's Spoke.

We did not try to maintain compatibility with upstream Spoke or test features and 
deployment strategies we did not use.
This README describes  only how to run this fork, see `ORIGINAL_README.md` for the original documentation.

## How the Warren campaign deployed Spoke

This fork was deployed to AWS Lambda and API Gateway using the [serverless framework](https://serverless.com/).
We used Aurora Postgres Serverless for our database and ElastiCache.

TODO[fuzzy]: description of the frontend deploy

## Notable changes from upstream spoke

- Removed support for Nexmo
- Removed support for Mailgun and custom SMTP servers in favor of SES
- Removed and renamed some env vars, notable TWILIO_API_KEY -> TWILIO_ACCOUNT_SID
- Removed asynchronous processing of Twilio responses
- Removed opt-out caching, which was incomplete in this fork
- Added support for using a different Twilio Messaging Service for each campaign
- Added external_id_type and state_code as top-level fields to campaign_contact
- Removed support for `zip` on campaign_contact
- Removed support for sqlite
- Reworked background jobs to run on Lambda, removed support for cron'd jobs
- Added slack log in
- Changed the invite flow
- Split the texter experience into separate components for initial sends and replies
- Removed bulk send
- Phone number management (see below)
- Reworked dynamic assignment experience, removed support for static assignment
- Moved message review to the campaign
- Added the concept of 'labels' for data collection through canned responses
- Added canned response upload from CSV
- Rewrote contact uploads to go through S3, which was required to upload large files while running on Lambda
- Started moving database access direct knex queries in the `db` package


## Running for the first time

Run `docker-compose up` to start the db.

Then, run `yarn run dev-migrate` and `yarn run create-test-db`

`./dev-tools/manage-messaging-services.js --command create --friendlyName <your test messaging service name> " --baseUrl <your ngrok url>`

`./dev-tools/buy-numbers.js --areaCode <any area code> --messagingServiceSid <your TWILIO_MESSAGE_SERVICE_SID> --limit 1`

### Running Locally

Open two terminal windows:

1. `docker-compose up`
2. `yarn run dev`

### Updating ngrok url for messaging service

`./dev-tools/manage-messaging-services.js --command update --baseUrl <YOUR_NGROK_URL> --sid <TWILIO_MESSAGE_SERVICE_SID>`

## Testing with auto-responders

If you'd like to test spoke without actually sending texts, you have a couple options.

### Skipping Twilio

We have an option to skip sending texts through Twilio and just simulating replies. In your .env, add the following two lines:

```
# This makes Spoke not send any real texts and just simulate a reply coming back
SKIP_TWILIO_AND_AUTOREPLY=1

# This makes Spoke randomly drop 60% of messages (so only ~40% of contacts will respond). This is based on
# a hash of the contact's phone number, so a particular contact will have a 60% chance of responding to
# the initial text, and then if they do respond they will continue to respond to future messages.
DROP_MESSAGE_RATIO=0.6
```

### Twilio Autoresponder

If you're working with the Twilio code itself, you may want to actually keep Twilio in the loop and
send real texts. To do this, replace `SKIP_TWILIO_AND_AUTOREPLY=1` with `OVERRIDE_RECIPIENT=+16173974753`.
This will send all texts, regardless of the contact's phone number, to `+16173974753`. This is a special
Twilio number that will auto-reply with whatever you sent it (try it out by sending a text to that number!)

We have some special handling in the code to still route texts correctly -- we append the contact's original
phone number to the outgoing message (so `foo` sent to `+16175555770` would become `[+16175555770] foo`), and
then the autoreply will keep that intact (so it would reply with `[+16175555770] foo (auto-reply)`). We then
parse out this tag in incoming messages and treat that message as a reply from `+16175555770`.

IMPORTANT NOTE: Twilio will block messages after 15 in a row go back and forth between two numbers
within 15 seconds. So if you use this setting to send more than 7 outbound messages, you'll
hit this limit and Twilio will stop sending messages. So the recommended flow is to switch
to this config just to do a small end-to-end test with a couple recipients (and probably
change DROP_MESSAGE_RATIO to 0 so you actually send all the messages).

### Generating test input data

To generate a CSV with, e.g., 1000 contacts, just run:

```
./dev-tools/generate-test-data.py 1000 > ./sample1000.csv
```

This will generate 1000 contacts with `555`-area-code phone numbers, black-hole email addresses,
`02141` as their ZIP, and a `fav_color` custom field.

In your .env, add the following to make phone numbers valid from the sample input data

```
SUPPRESS_PHONE_VALIDATION=1
```

### Working with a reasonable batch size for dynamic assign

The default size for batch assign is 300, but when testing you can change the batch size to a more reasonable tested number by setting in the .env file.

```
DYNAMIC_ASSIGN_MAX_BATCH_SIZE=10
```

### Phone number provisioning

There are two ways to provision Twilio phone numbers and Messaging Services in the Warren fork of Spoke:

- Using the `campaignPhoneNumbers` feature, which provisions Messaging Services for
  campaigns on-the-fly and adds numbers from a shared inventory. This feature allows
  Spoke to scale beyond the limits of a single Messaging Service (400 numbers or about
  80,000 texts a day). Numbers and Messaging Services are managed by the application.

- Using the global Messaging Service set by the TWILIO_MESSAGE_SERVICE_SID env var.
  We also support setting a message_service_sid in organization features, but this is
  not used currently and may be removed in the future. The messaging service needs to
  be created and managed manually. See `dev-tools/` for scripts to help with this.

The rest of this section describes `campaignPhoneNumbers` feature.

#### Enabling `campaignPhoneNumbers`

You must be an OWNER of an organization to enable the `campaignPhoneNumbers`.
There is no UI for this but it can be done through a mutation in Graphiql:

```
mutation {
  enableCampaignPhoneNumbers(organizationId: "12345") {
    campaignPhoneNumbersEnabled
  }
}
```

#### The phone number inventory

`campaignPhoneNumbers` relies on an inventory of phone numbers managed by the Spoke application.
Managed phone numbers are tracked in the `twilio_phone_number` table and can be in one
of three states:

- `AVAILABLE`: available to be claimed by a new campaign
- `RESERVED`: claimed by an unstarted campaign. Reservations expire to prevent
  campaigns from holding numbers without using them.
- `ASSIGNED`: claimed by an active campaign and added to the Twilio Messaging Service
  created for that campaign.

The phone number inventory is shared by all organizations in a Spoke instance. This was
desirable for Warren for President but might not be suitable for all Spoke deployments.

#### Campaign creation

When the `campaignPhoneNumbers` feature is enabled, admins will have to configure
phone numbers when building a campaign. The "Phone Numbers" section in the
campaign creation flow allows admins to select the area codes they want to use from
the list of available phone numbers in the inventory.

This also enforces a ratio of 150 contacts per number to stay well within the
recommended limit of 200 texts/day for P2P texting. A campaign cannot start until enough
numbers have been reserved.

By design, an admin cannot purchase numbers or make modifications to Twilio while
editing a draft campaign. Starting a campaign creates a Messaging Service and
transfers already-purchased numbers to it. Once a campaign is started, the Messaging
Service and phone number configuration cannot be changed.

#### Purchasing numbers for the inventory

To buy more numbers and make them available to campaign creators, SUPERADMINS can
run a mutation in Graphiql.

Example: purchase 3 numbers in the 952 area code:

```
mutation {
  buyNumbers(areaCode: "952", limit: 3) {
    id
  }
}
```

_IMPORTANT: the `twilio_phone_numbers` table and buy-numbers background job are not
intended to be used with the global Twilio messaging service. Instead, use
`dev-tools/buy-numbers.js` to buy numbers if you don't have `campaignPhoneNumbers`
turned on._

#### Voice responses

To override the default voice response for Spoke-managed numbers set the TWILIO_VOICE_URL
env var. We use a simple TwiML Bin response:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Play>https://ew-spoke-public.s3.amazonaws.com/2020-01-12_DTC_Text+Incoming+Message.wav</Play>
</Response>
```

#### Sticky sender

The `campaignPhoneNumbers` feature was designed to avoid texting contacts with the same number from
different campaigns. This solves two problems:

1. It avoids threading messages from different campaigns/volunteers on the contact's phone
2. It allows us to route texts correctly for contacts who receive messages from more
   than one campaign. By comparison, Spoke will route incoming messages to the campaign
   that texted a contact most recently when using a shared Messaging Service.

Note that phone numbers are re-used, so it is possible for a contact to receive messages
from the same number from different campaigns, though it is unlikely at scale.
It is not possible, however, for a number to be used by two active campaigns simultaneously,
so we still manage to avoid routing problems.

# License

Spoke is licensed under the MIT license.
