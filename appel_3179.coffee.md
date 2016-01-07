    pkg = require './package'
    @name = "#{pkg.name}:appel_3179"
    debug = (require 'debug') @name
    seem = require 'seem'
    Promise = require 'bluebird'
    seconds = 1000

    @include = seem (ctx) ->

      debug 'Start',
        direction: @session.direction
        dialplan: @session.dialplan
        country: @session.country
        destination: @destination

Should we intercept on the global format (3303179) or the local format?
This needs to be global so that it works for Centrex as well.

However do not assume we run inside tough-rate. Only assume useful-wind for now. This means we have access to:
- cfg
- call
- data
- source
- destination
- req.header()
- session
- action()

      return unless @session.direction is 'egress'
      return unless @session.dialplan is 'national'
      return unless @session.country is 'fr'
      return unless @destination is '3179'

Prevent further processing.

      @session.direction = null

      @statistics.emit 'rio-request', source:@source

      get_rio_index = seem (rios) =>
        debug "get_rio_index", {rios}
        @pencil.clear()
        # FIXME set dtmf_min_length
        if rios.lengh > 1
          yield @pencil.play 'welcome_internal'
          yield @pencil.play 'enter_number_first'
          for number,rio in rios
            yield @pencil.play 'for_number'
            yield @pencil.spell number
            yield @pencil.playback "voicemail/vm-press"
            yield @pencil.playback "digits/#{i+1}"
        else
          @session.dtmf_buffer = '1'

        yield Promise.delay 3*seconds
        if @session.dtmf_buffer.length is 0
          get_rio_index()
        else
          index = parseInt(@session.dtmf_buffer)-1
          @session.number = rios[index].number
          @session.rio = rios[index].rio
          @pencil.clear()

      retrieve_user = (number) ->
        debug "retrieve_user", {number}
        Promise.resolve
          rios: {
            '0972222713': 'OO Q RRRRR CCC'
          }
          email: 'stephane@shimaore.net'
          nom: 'Stéphane Test'
          addresse_de_facturation: '42 rue des Glycines\nAppt 42\n99123 Gogrville'

      @session.doc = yield retrieve_user(@source).catch seem (error) =>
          @statistics.emit 'rio-failed', source:@source
          yield @action 'respond', 503
          @end

      unless @session.doc.rios?
        debug 'No RIOs'
        yield @pencil.spell 'BP93'
        yield @action 'hangup'
        return

      rios = ({number,rio} for own number,rio of @session.doc.rios)
      index = yield get_rio_index rios
      yield @pencil.play 'the_rio'
      yield @pencil.spell @session.number
      yield @pencil.play 'is'
      yield @pencil.spell @session.rio
      yield @pencil.play 'dont_cancel'


Tools to send out
=================

      send_sms = seem (recipient,text) =>
        debug 'send_sms', {recipient,text}
        yield @pencil.playback 'ivr/ivr-thank_you_alt'
        yield @action 'hangup'

      send_email = seem (recipient,html) =>
        debug 'send_email', {recipient,html}
        yield @pencil.playback 'ivr/ivr-thank_you_alt'
        yield @action 'hangup'

      send_snailmail = seem (recipient,address) =>
        debug 'send_snailmail', {recipient,address}
        yield @pencil.playback 'ivr/ivr-thank_you_alt'
        yield @action 'hangup'


Destination
===========

      @pencil.clear 1

Send via SMS
------------

We don't have a proper list of customer mobile numbers at this time. Skip until we can provide this.

      ###
      if @session.doc.mobile?
        yield @pencil.play 'sms_known'
        yield @pencil.spell @session.doc.mobile
        yield @pencil.playback "voicemail/vm-press"
        yield @pencil.playback "digits/1"

      yield @pencil.play 'sms_unknown'
      yield @pencil.playback "digits/2"
      ###

Send via email
--------------

      if @session.doc.email?
        debug 'Send via email'
        yield @pencil.play 'email'
        yield @pencil.spell @session.doc.email
        yield @pencil.playback "voicemail/vm-press"
        yield @pencil.playback "digits/3"

Send via postmail
-----------------

      yield @pencil.play 'snailmail'
      yield @pencil.playback "voicemail/vm-press"
      yield @pencil.playback "digits/4"

      yield Promise.delay 10*seconds

      sms_text = """
        Le RIO associé au numéro #{@session.number[0...6]}XXXX est #{@session.rio}.
      """

      choice = @pencil.clear()
      switch choice
        when '1'
          if @session.doc.mobile?
            send_sms @session.doc.mobile, sms_text
        when '2'
          yield @pencil.playback 'ivr/ivr-please_enter_the_phone_number'
          yield Promise.delay 16*seconds
          send_sms @session.dtmf_buffer, sms_text
        when '3'
          send_email @session.doc.email, """
            <p>
            Veuillez trouver ci-dessous la liste des RIO fixes associés à votre compte.
            </p>
            <table>
              <tr><th>Numéro de téléphone</th><th>RIO</th></tr>
              #{ ["<tr><td> #{e.number} </td><td> #{e.rio} </td></tr>\n" for e in rios].join '' }
            </table>
          """
        when '4'
          send_snailmail @session.doc.nom, @session.doc.address_de_facturation

        else
          debug 'No valid choice was made, aborting.'
          yield @pencil.playback 'ivr/ivr-thank_you_alt'
          yield @action 'hangup'
