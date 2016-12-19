    pkg = require './package'
    @name = "#{pkg.name}:vacation"
    debug = (require 'debug') @name

    assert = require 'assert'
    seem = require 'seem'

    moment = require 'moment-timezone'
    day_of = (t) -> t.clone().startOf 'day'

    @include = seem ->
      return unless @session.direction is 'egress'

      debug 'Ready'

      is_on_vacation = seem (vacation) =>
        debug 'is_on_vacation'

Vacation settings

        return false unless vacation?

* doc.local_number.vacation.start (string) either a date or timestamp (in ISO8601 format, local time or with timezone) that specifies the earliest point in time the user is to be considered on vacation
* doc.local_number.vacation.end (string) either a date or timestamp (in ISO8601 format, local time or with timezone) that specifies the latest point in time the user is to be considered on vacation

        now = moment.tz @timezone()

        if vacation.start?
          start = moment.tz vacation.start, @timezone()
          return false unless start.isBefore now

        if vacation.end?
          end = moment.tz vacation.end, @timezone()
          return false unless now.isBefore end

The user is on vacation. Now let's try to say something intelligent about it.

        today = day_of now

        yield @pencil.play 'vacation-person_is_on_vacation'

        if start?
          start_day = day_of start

If the start date is today then simply play the hour.

          if start_day is today
            yield @pencil.play 'vacation-since'
            yield @action 'phrase', "say-time,#{start.format 'HH:mm'}"

TODO: yesterday, etc.

          else
            yield @pencil.play 'vacation-since_day'
            yield @action 'phrase', "say-day-of-month,#{start.format 'DD'}"
            yield @action 'phrase', "say-month,#{start.format 'MM'}"

        if end?
          end_day = day_of end

          if end_day is today
            yield @pencil.play 'vacation-person_will_return'
            yield @pencil.play 'vacation-at'
            yield @action 'phrase', "say-time,#{end.format 'HH:mm'}"
          else
            yield @pencil.play 'vacation-person_will_return'
            yield @pencil.play 'vacation-on'
            yield @action 'phrase', "say-day-of-month,#{end.format 'DD'}"
            yield @action 'phrase', "say-month,#{end.format 'MM'}"

        true

Assume we are in a `huge-play` client/egress context, and @session.number contains the document.

      if yield is_on_vacation @session.number.vacation
        @session.direction = 'vacation'
        debug 'Hangup'
        yield @action 'hangup'
        return
