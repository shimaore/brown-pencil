    pkg = require './package'
    @name = "#{pkg.name}:appel_3179"
    debug = (require 'debug') @name
    seem = require 'seem'
    Promise = require 'bluebird'
    seconds = 1000

    bucket = require 'glorious-bucket'

    @include = seem ->

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

      {send_sms,send_email,send_snail_mail,retrieve_user} = bucket @cfg

Prevent further processing.

      @session.direction = null

      @statistics.emit 'rio-request', source:@source

      yield @pencil.play 'welcome_internal'

      get_rio_index = seem (rios) =>
        debug "get_rio_index", {rios}
        @pencil.clear()
        # FIXME set dtmf_min_length
        debug 'get_rio_index length', rios.length
        if rios.lengh > 1
          @session.dtmf_buffer = ''
          yield @pencil.play 'enter_number_first'
          for v,i in rios
            debug 'get_rio_index record', v
            yield @pencil.play 'for_number'
            yield @pencil.spell v.number
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
          return index

      debug 'retrieve user', @source

      @session.doc = yield retrieve_user(@source)
        .catch seem (error) =>
          debug "retrieve user: #{error.stack ? error}"
          @statistics.emit 'rio-failed', source:@source
          yield @action 'respond', 503
          @end
          null

      debug 'retrieve user returned', @session.doc
      return unless @session.doc?

      unless @session.doc.rios?
        debug 'No RIOs'
        yield @pencil.spell 'BP93'
        yield @action 'hangup'
        return

      rios = ({number,rio} for own number,rio of @session.doc.rios)
      index = yield get_rio_index rios
        .catch (error) ->
          debug "get_rio_index failed #{error.stack ? error}"
          get_rio_index rios
      yield @pencil.play 'the_rio'
      yield @pencil.spell @session.number
      yield @pencil.play 'is'
      yield @pencil.spell @session.rio
      yield @pencil.play 'dont_cancel'


Destination
===========

      @pencil.clear 1

Send via SMS
------------

      if @session.doc.mobile?
        yield @pencil.play 'sms_known'
        yield @pencil.spell @session.doc.mobile
        yield @pencil.playback "voicemail/vm-press"
        yield @pencil.playback "digits/1"

      yield @pencil.play 'sms_unknown'
      yield @pencil.playback "voicemail/vm-press"
      yield @pencil.playback "digits/2"

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
            yield send_sms @session.doc.mobile, sms_text
        when '2'
          yield @pencil.playback 'ivr/ivr-please_enter_the_phone_number'
          yield Promise.delay 16*seconds
          yield send_sms @session.dtmf_buffer, sms_text
        when '3'
          yield send_email @session.doc.email, rios
        when '4'
          yield send_snailmail @session.doc.nom, @session.doc.address_de_facturation, rios

        else
          debug 'No valid choice was made, aborting.'

      yield @pencil.playback 'ivr/ivr-thank_you_alt'
      yield @action 'hangup'
