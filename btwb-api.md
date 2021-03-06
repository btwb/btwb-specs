BTWB Api Overview
=================

This document describes the BTWB API for 3rd Party Applications.

The BTWB API is not intended to open up the full capabilities
of the BTWB Platform. This API is intended only to provide a
subset of capabilities, as determined necessary, to support the
BTWB Partners in implementing their specific needs.

Note that the BTWB API is only available to pre-approved
BTWB Partners, and is not open to the general public.

Please note that this documentation may be sparse, but will
hopefully provide BTWB Partners enough information to
develop their integrations with the BTWB Platform.


Terminology
===========

Application - The BTWB Partner's software known by the BTWB Platform.

Agent - The process that has the power/authority to act on behalf
  of the Application. Each Agent is tied to only one Application.
  Each Agent is identified by, and is authenticated with, its
  Agent Access Token.

Agent Access Token - The case sensitive opaque string of digits,
  letters and special characters that make up a Bearer token,
  which the Agent uses to authenticate itself to the BTWB API.

Access Key - The representation of a set of auth Scopes, which enables
  a set of permissions, over Resources owned by the Resource Owner
  granted explicitly to a specific Application.
  Synonymous with Oauth2 Authorization Grant

Access Key Token - The case sensitive opaque string of digits,
  letters and special characters that represent the Access Key.
  Synonymous with the Oauth2 Bearer Access Token.


HTTP Authorization
==================

Requests made to the BTWB API must provide both the HTTP Authorization header
and the X-Agent header. These credentials are both the Gym API credentials,
and the Application's own Agent credentials.

The Agent credentials authenticates the Application making the request;
and the Authorization credentials authenticate that the request being
made is for the Resource Owner (ie, Gym) and has specific permissions
defined by the Access Key's scope
authorized the specific Application with permissions granted by the scope
of said token.

In the structure of:

    Authorization: Bearer {GYM_ACCESS_KEY_TOKEN}
    X-Agent: btwb {API_AGENT_SECRET KEY}

Example HTTP Request Headers:

    Authorization: Bearer _d00/k_2bwsrfdvG9c8EaP1HUHKT_
    X-Agent: btwb _co0/k_-vqnOmrhOEDtPPmrgvajnd

On Authentication or Authorization failure, BTWB API will return one
of the following HTTP Status Codes:

    401 - Failed Authorization Keys. For example,

        {
          "@type":"https://btwb.info/cards/Error",
          "name":"Api::Errors::Unauthorized",
          "content":"Invalid X-Agent Token"
        }

    403 - Authenticated, but the request was Unauthorized



Gym General API
===============


Ping
----

This allows the Application to make a simple HTTP GET request
with their own X-Agent credentials along with the Gym's own
Access Key Token to verify that the credentials authenticate
properly.

HTTP METHOD: GET
URI Template: `/gyms/{gym_id}/ping`

HTTP JSON Response:

    200 - Successful authentication

        {
          "@type": "https://btwb.info/events/Pong",
          "gym_id": {gym_id}
        }

    401 - Failed Authorization Keys. For example,

        {
          "@type":"https://btwb.info/cards/Error",
          "name":"Api::Errors::Unauthorized",
          "content":"Invalid X-Agent Token"
        }

    403 - Authenticated, but the request was Unauthorized


Gym Linked Memberships / Member Sync
====================================


Link and Synchronize Member
---------------------------

This is an idempotent operation that allows the Application to both Link
their own Member with a BTWB Member. BTWB API will determine how to
update the BTWB state based on the input and the currently known BTWB state.

HTTP Method: POST
Uri Template: `https://api.prod.btwb.com/gyms/{gym_id}/memberships`
Post Body Params:

    correlation_key - A unique identifier provided by the Application that
      represents a member account within the Application's own domain.
    email - Email address of the Member to sync
      This field is required for correlation_keys that have not been
      linked in BTWB. This field is *not* always required, but SHOULD
      be sent every time.
    first_name - Member's First Name
    last_name - Member's Last Name
    birthday - 'YYYY-MM-DD'
    gender - `Male` or `Female` case sensitive
    street
    city
    state
    postal_code
    country

How BTWB Handles Sync&Link Requests:

* A GymLinkMembership is always created with this request, which
  ties together the following:
    1. The Application
    2. The Gym
    3. An Existing BTWB Member (If BTWB Member creation failed,
      future calls will resolve the linking properly).
* If there already exists a known GymLinkedMembership for the
  correlation_key, then BTWB API will attempt one of the following:
    1. If the Linked Member currently belongs to the requesting Gym,
      then all member's information fields are updated.
      The EXCEPTION is that the `email` is never updated.
    2. If the Linked Member does NOT belong to the requestion Gym,
      then an invitation is sent to the Member to join the Gym.
* If the correlation_key is unknown to BTWB, then BTWB will:
    1. Attempt to find an existing BTWB Member with that Email.
      a. Found Members that also belong to the requesting Gym
        will have its information updated. Again, all but email.
      b. Found Members that do NOT belong to the requesting Gym
        will be sent a gym invitation.
    2. Create a brand new BTWB Member (if email was not found)
      and then add that member to the Gym.
      In cases where the Gym has reached its member threshold, the
      newly created member will be in MainSite, and will be given
      a gym invitation request. The invitation can be reclaimed once
      the Gym upgrades its account.
      Please note that created Members are set to a random password.
      In order for these members to log in, they must reset their password
      via the password reset links sent to their email.

HTTP JSON Responses:

    200 - Success

        {
          "@type":  'https://btwb.info/events/gyms/LinkedMembershipUpdated'
          "content": "A Success Message",
          "correlation_key": "applications_correlation_key",
          "profile_url": "http://btwb.com/_XXX"
        }

      `content` is a free form text response. You may return this
        information directly to the end user.

      `profile_url` is a URL that will redirect to the member's correct btwb profile page.

      Success messages may be, but not limited to:
        "Member was updated, except for email"
        "Member was updated"
        "Member is up-to-date, except for email"
        "Member is up-to-date"
        "Member was invited to the gym"
        "Member was created, and was added to the gym"
        "Member was created, but was invited to the Gym. Please upgrade your gym plan to allow more members."

    500 - Error in request: Eg. Validation Errors in creating a member

        {
          "@type":  "https://btwb.info/cards/Error",
          "content": "An Error Message"
        }


Kick Member from Gym
--------------------

HTTP Method: POST
URI Template: `https://api.prod.btwb.com/gyms/{gym_id}/memberships/{correlation_key}/kick`

HTTP JSON Responses:

    200 - Success

        {
          "@type": "https://btwb.info/events/gyms/LinkedMemberKicked",
          "content": "Member identified by {{correlation_key}} removed from gym",
          "correlation_key": "my_correlation_key"
        }

    500 - Server Error

        {
          "@type":  "https://btwb.info/cards/Error",
          "content": "An Error Message"
        }

      `content` is a free form text response. You may return this
        information directly to the end user.

      Error messages may be, but not limited to:
        "Gym Linked Membership not found"
        "Gym Linked Membership does not reference a member"
        "Unable to save to the database"


Unlink from Gym
---------------

HTTP Method: DELETE
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/memberships/{correlation_key}

This will delete the Gym Linked Membership for your Application's mapping between the
GymId and the CorrelationKey. Attempting to Sync the member again may have unintended
consequences. Eg. If the member has changed emails in either BTWB Platform or the Application's
own platform (a linking may create a new member, instead of inviting or editing linked members)

WARNING - USE ONLY IN RARE OCCASSIONS.

HTTP JSON Responses:

    200

        {
          "@type": "https://btwb.info/events/gyms/LinkedMembershipDeleted",
          "correlation_key": "my_correlation_key"
        }

    500 - Server Error

        {
          "@type":  "https://btwb.info/cards/Error",
          "content": "An Error Message"
        }

      `content` is a free form text response. You may return this
        information directly to the end user.

      Error messages may be, but not limited to:
        "Unable to save to the database"



Gym Membership Checkins
=======================

This allows the Application to manage member checkins on behalf of a gym.


List Gym Checkins
-----------------

HTTP Method: GET
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/checkins

Query Params:

  * per_page - [default 100] How many checkins to list on GET
  * page - [default 1] Pagination Offset.


Checkin Member
--------------

Creates a Checkin for a member.

HTTP Method: POST
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/checkins

Post Body Params:

    correlation_key - A unique identifier provided by the Application that
      represents a member account within the Application's own domain.
    checkin_at - [optional] ISO 8601 formatted string of checkin date and time

HTTP JSON Responses:

    200

        {
          "checkins": [
            {
              "id": "71712fcb-39f2-4688-8b9f-35834047617c",
              membership_correlation_key: "my_correlation_key",
              checkin_at: "2016-06-10T11:45:43-04:00",
              created_at: "2016-06-10T11:45:43-04:00"
            }
          ]
        }

Lookup Existing Checkin
-----------------------

HTTP Method: GET
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/checkins/{checkin_id}

HTTP JSON Responses:

    200

        {
          "checkins": [
            {
              "id": "71712fcb-39f2-4688-8b9f-35834047617c",
              membership_correlation_key: "my_correlation_key",
              checkin_at: "2016-06-10T11:45:43-04:00",
              created_at: "2016-06-10T11:45:43-04:00"
            }
          ]
        }

Delete Checkin
--------------

Deletes a given checkin

HTTP Method: DELETE
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/checkins/{checkin_id}

Post Body Params:

    correlation_key - A unique identifier provided by the Application that
      represents a member account within the Application's own domain.

HTTP JSON Responses:

    200

        {
          "checkins": [
            {
              "id": "71712fcb-39f2-4688-8b9f-35834047617c",
              membership_correlation_key: "my_correlation_key",
              checkin_at: "2016-06-10T11:45:43-04:00",
              created_at: "2016-06-10T11:45:43-04:00"
            }
          ]
        }


Gym Tracks And Wods
===================

This allows the Application to look up a Gym's Tracks and list any
specific track's TrackEvents.

List Gym Tracks
---------------

Obtain the list of all the Gym's tracks.

HTTP Method: GET
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/tracks

Query Params:

  NONE

HTTP JSON Responses:

    200

        {
          "tracks": [
            {
              "id": 1,
              "name": "My Gym Track",
              "open": true,
              "created_at": "2016-06-10T11:45:43-04:00",
              "updated_at": "2016-06-10T11:45:43-04:00"
            }
          ]
        }

List Gym Track Events
---------------------

Lists the Gym's planned track events for a given track, on
a given date. The possible track events returned include
Workouts, planned rest days and weigh ins.

HTTP Method: GET
URI Template: https://api.prod.btwb.com/gyms/{gym_id}/track_events

Query Params:

  * track_id - [required] The track id whose track events to look up.
  * date - [default today] The YYYY-MM-DD date string to look up.

HTTP JSON Responses:

    200

        {
          "track_id": 1234,
          "date": "2019-01-01",
          "track_events": [...]
        }

    track_event objects:

        {
          "id": 9999,
          "event_date": "2019-01-01",
          "track_id": 1234,
          "track_name": "My Main Track",
          "task_type": "Workout",
          "title": "Optionally Set Title for the Wod",
          "notes": "Optionally Set Notes for the Wod",
          "position": 1,
          "section": "main",
          "workout_name": "Fran",
          "workout_description": "21-15-9 ..."
        }

Coaching
========

List Programs
-------------

Returns a list of the Coaching Programs for the Authenticating
Coaching Team.

HTTP Method: GET
URI Template: https://api.prod.btwb.com/coaching/programs

Query Params:

  * per_page - [default 100] How many checkins to list on GET
  * page - [default 1] Pagination Offset.


Get Program
-----------

Returns a the Coaching Program by id for the Authenticating
Coaching Team.

HTTP Method: GET
URI Template: https://api.prod.btwb.com/coaching/programs/{id}

URL Path Params:
  * id - The coaching program id.

Create Subscription
-------------------

Subscribes either a Gym or a Member to a Coaching Program.
This action is only available to Coaching Teams with a valid
Api Access Key granted to partnered Api Applications.

HTTP Method: POST
Uri Template: `https://api.prod.btwb.com/coaching/subscriptions`

Post Body Params:

    coaching_program_id - [required] The id of the program.

    subscriber_type - [required] "Gym" or "Member", this lets Btwb
      know the type of subscription to create.

    gym_id - [optional] Identifies the gym that is to be subscribed.
      If this param is given, it will supercede `member_id` and `email`
      params.

    member_id - [optional] Identifies the Btwb member account to be
      subscribed. If this param is given, it will supercede `email`
      param matching.

    email - [optional] Email address of the Member to sync. We
      will first attempt to find the Btwb member account with the given
      email. If not found, we will create a new member account
      with the remaining member informational params.

    gender - [optional] *Required* when a member with the given
      email is not found, thus we will create a new member account.
      `Male` or `Female` case sensitive.

    first_name - [optional] To be created member's First Name.
      Defaults to 'FName'

    last_name - [optional] To be created member's Last Name.
      Defaults to 'LName'

    birthday - [optional] To be created member's dob in form of 'YYYY-MM-DD'.
      Defaults to '1900-01-01'.

    street - [optional] To be created member's street address.
    city - [optional] To be created member's city.
    state - [optional] To be created member's state.
    postal_code - [optional] To be created member's postal code.
    country - [optional] To be created member's country.

HTTP JSON Responses:

    200

        {
          "@type": "https://btwb.info/events/coaching/SubscriptionCreated",
          "subscriber_url": "http://www.btwb.com/members/1",
          "subscription": {
            "id": 1,
            "subscriber_id": 1,
            "subscriber_type": "Gym",
            "subscriber_name": "CrossFit Kinnick",
            "subscriber_email": "support@crossfitkinnick.com",
            "coaching_program_id": 1,
            "coaching_program_plan_id": null,
            "created_at": "2017-08-10T13:18:29.435-07:00",
            "canceled_at": null,
            "ended_at": null
          }
        }

List Subscriptions
------------------

Returns a list of active Coaching Programs Subscriptions for the
Authenticating Coaching Team.

This means any Gym or Member subscription for any program for
the given Coaching Team whose ended_at is not set (or in the future).

HTTP Method: GET
URI Template: https://api.prod.btwb.com/coaching/subscriptions

Query Params:
  * email - [optional] Email address of subscriber.
      If a found member is the Gym Subscription Owner in Btwb,
      then we also return the Gym's Subscriptions. Otherwise,
      It's just the list of subscriptions owned by the individual.
  * per_page - [default 100] How many checkins to list on GET
  * page - [default 1] Pagination Offset.

Get Subscription
----------------

Lookup a subscription by id.

HTTP Method: GET
URI Template: https://api.prod.btwb.com/coaching/subscriptions/{id}

URL Path Params:
  * id - The subscription id.

Cancel Subscription
-------------------

Ends the given subscription for the given Authenticating Coaching Team.

HTTP Method: POST
URI Template: https://api.prod.btwb.com/coaching/subscriptions/{id}/cancel

Post Body Params:

  * end_immediately - [optional] If set (ie, true or 1), then we will cancel
      the given subscription with an immediate end date of NOW.
      Otherwise, the subscription will end on the end of the current
      billing cycle.


