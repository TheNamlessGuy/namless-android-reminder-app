# What is the app?
This is an app to help you remember to do things, whether through calendar events, or through timers.

# General
A list of general rules and ideas

- All object format proposals are there more to understand what features are required, not necessarily reflecting the end result
- All DATETIME attributes should be stored and fetched as UTC, and only converted to local time when displayed to the user, or when the system calls require it
- All RELTIME attributes should be strings formatted something like `-4D` for "4 days earlier" and `+3D5h2m` for "three days, 5 hours, and 2 minutes later"
  - Always starts with `-` or `+`
  - Should support years (`Y`), months (`M`), days (`D`), hours (`h`), and minutes (`m`)
  - Always have to be in correct order (Y->M->D->h->m) - validated on save, not on read. If this rule is broken, the result _should_ be broken
  - No zero values - `0D`, for example. No offset = `null`
  - All these attributes should also have a `<attribute>_cache` that's a DATETIME with the offset applied
    - If the offset is between `created_at` (1970-01-01 00:00:00) and offset `created_at_offset` (`+1h`), `created_at_offset_cache` should be "1970-01-01 01:00:00"
    - One cache per DATETIME the offset is relative to. When multiple caches exist, the cache names should be formatted as `<attribute>_cache_for_<datetime attribute>`
    - These are _cached_ and _must_ be kept up to date with the DATETIME attribute it is relative to, and changes to the offset itself
- "Reminder" can mean both an alarm and a regular notification, unless specified otherwise
- Attributes whose names end in `?` should be considered "a rough idea" or "unsure if this is something we actually want"

# Events
Events are the cornerstone of the entire app function.

They are displayed in a calendar format.

## Event object proposal 
- id - The ID of the event
- active - A boolean indicating the event is active (should fire) or not
- reminder_type - An enum: `ALARM` or `NOTIFICATION`
- fire_at - The DATETIME on which the event should fire
- auto_dismissal_offset - An optional RELTIME at which the event should automatically be dismissed. If none is set, the event lives until manually dismissed
- auto_dismissal_type - An enum with values `BASIC`, and `NEXT` - if next_id is set and `NEXT` is the type, the auto-dismiss should act as if the user pressed the button associated with the next_id
- title - A string that's the title of the event
- description - An optional string that's complementary to the title
- dismissal_reason - An enum value storing `AUTO` if the event was dismissed automatically, `USER` if it was manually dismissed by the user, or `null` if it hasn't been dismissed yet
- is_swipeable - Whether the resulting reminder should be swipeable
- has_notification_dismissal - Whether the event can be dismissed from the reminder (most likely through some "OK" or "Dismiss" button)
- is_snoozeable - Whether the event should be snoozeable ("Refire again at RELTIME" - the reltime being set in the options menu, and being overridable per event)
- next_id - An optional ID pointing to another event that can be activated from this event - Used for something like "I had to dismiss the morning workout reminder due to meetings, activate the afternoon workout reminder"

`has_notification_dismissal`, `is_snoozeable`, and `next_id` should all, if active in whatever way they are activated, result in unique buttons on the reminder.  
`is_swipeable` results in the notification being swipeable (if it isn't a fullscreen one) - sort of a button, but not really.

All of `is_swipeable`, `has_notification_dismissal`, `is_snoozeable`, and `next_id` are applicable to all types of reminders. `is_swipeable` is however only relevant in the case where the alarm is NOT fullscreen - such as if the user is watching a video or some such. The default values for these should also differ based on the type of the event.

If an event has auto-dismissal and is snoozeable, the dismissal offset should be recalculated each time the event fires.

# Reminders
Reminders remind you of an event - before or after the event itself has fired.

They should be considered the same as events, but always have:
- reminder_type = `NOTIFICATION`
- is_swipeable = true
- has_notification_dismissal = true

and all other features turned off.

## Reminder object proposal
- id - The ID of the reminder
- event_id - The ID of the event it is a reminder of
- fire_offset - A RELTIME for when the reminder should be fired, relative to event.fire_at
- dismissal_reason - Same purpose as the same attribute on the event object. Valid values: `USER`, `null`

# Pattern
A pattern is a series of events that are generally repeatable. A pattern can be a pattern over a day, a week, a month, or a year. A user can have as many patterns as they want.

If a pattern or a pattern event is updated, all the events generated from the pattern should follow suit immediately (so they may need a `generated_from_pattern_event_id` attribute, or some other way of knowing that relation).

The use case for this are things like:
- Having two daily "Remember to brush your teeth!" reminders
- Having yearly birthday reminders
- Reminders to pay your bills each month

Patterns are generated into real events X time instances in advance
- "time instance" is relative to the pattern type - X days, X weeks, X months, etc...
- Each type should have its own default X in the settings
- Each pattern should be able to override this default
- The defaults should be:
  - 7 DAYS
  - 2 WEEKS
  - 2 MONTHS
  - 2 YEARS

Deactivating a pattern should delete (not deactivate) all future events generated from the pattern.

Events generated from a pattern should not be individually editable.

## Pattern object proposal
- id - The ID of the pattern
- type - DAY, WEEK, MONTH, YEAR - probably an enum of some sort
- active - Whether the pattern is active

## Pattern event object proposal
Basically a superset of the event object.

- fire_at_offset - replaces `fire_at`. A RELTIME that's relative to when the pattern is applied - "00:00:00" for DAY patterns, "00:00:00" on the Monday for WEEK patterns, and so on, from which the resultin `fire_at` on the generated event is calculated.
- pattern_id - The ID of the pattern this is relative to

`next_id` should point to another pattern event under the same pattern, and the ID should be updated to the actual generated event upon generation.

This is the only object where we need to take summer/winter time into account - if I say "the alarm goes at 07:00", it should _always_ fire at 07:00, regardless of if I move timezones, or if the timezone shifts relative UTC (i.e. summer/winter time).

# Timer
Instead of a "remind me on...", this is a "remind me in...". Egg timer type things - "The washing machine is done in 2 hours"

Should just be displayed in a long list, and don't have to be kept around once they've fired. They don't need a field denoting if it's been dismissed or not - once fired and dismissed, it should be deleted from the database.

Timers are basically a simple subset of alarm events.

## Timer object proposal
- id - The ID of the timer
- created_at - A DATETIME of when the timer was created
- fire_offset - A RELTIME relative `created_at` for when the event should trigger
- title - An optional string for what the timer is for
- is_swipeable - Whether the resulting reminder should be swipeable
- has_notification_dismissal - Whether the timer can be dismissed from the reminder (most likely through some "OK" or "Dismiss" button)
- is_snoozeable - Whether the timer should be snoozeable ("Refire again at RELTIME" - the reltime being set in the options menu, and being overridable per event)

# Category
Each event should be able to be part of a category - "Work", one for each of your kids, "DnD". Any event should be able to have as many categories as it wants, and vice versa.

A category has a related color. Events should be colored relative to their categories.

A category should be toggleable on and off, which should toggle all the related events on and off
- Use case is something like "I don't want work notifications during my vacation, but I also don't want to have to recreate all the events when I get back to work"
- Pattern generation needs to take this into account - determines if the generated event is active or not.
- Category active status supersedes event active status - if the category is off, so is the event. You can't change the event toggle without changing the category one.
- If _any_ of an events categories are off, the event is off.

## Category object proposal
- id - The ID of the category
- title - The title of the category
- color - The HSL value that's associated with the category
- active - Whether the category (and thus all its events) are active

# Other functions
- The app should be able to back up its data to:
  - Local storage
  - Mega.nz
  - Dropbox?
- The app should be able to fetch data (readonly) from:
  - Google calendar?
  - Office calendar
- The app checks that it has the correct permissions and so on, or it won't allow you to use it
  - "Exact alarms" turned on
  - "Notifications" turned on
  - "Battery optimizations" disabled (this one shouldn't be blocking, but it should be shown to the user as a warning)