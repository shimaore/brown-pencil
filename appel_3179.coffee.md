    pkg = require './package'
    @name = "#{pkg.name}:appel_3179"
    debug = (require 'debug') @name
    seem = require 'seem'
    Promise = require 'bluebird'
    seconds = 1000

    bucket = try require 'glorious-bucket'

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

      {send_sms,send_email,send_snailmail,retrieve_user} = bucket @cfg

Prevent further processing.

      @session.direction = null

      @statistics.emit 'rio-request', source:@source

      yield @pencil.play 'welcome_internal'

      get_rio_index = seem (rios) =>
        debug "get_rio_index", {rios}
        @pencil.clear()
        # FIXME set dtmf_min_length
        debug 'get_rio_index length', rios.length
        if rios.length > 1
          debug 'Listing RIOs'
          @session.dtmf_buffer = ''
          yield @pencil.play 'enter_number_first'
          for v,i in rios when v.rio?
            debug 'get_rio_index record', i,v
            yield @pencil.play 'for_number'
            yield @pencil.spell v.number
            yield @pencil.playback "voicemail/vm-press"
            yield @pencil.playback "digits/#{i+1}"
        else
          debug 'Only one RIO available'
          @session.dtmf_buffer = '1'

        yield Promise.delay 3*seconds
        if @session.dtmf_buffer.length is 0
          get_rio_index()
        else
          debug 'User selection', @session.dtmf_buffer
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
        debug 'Announce mobile', @session.doc.mobile
        yield @pencil.play 'sms_known'
        yield @pencil.spell @session.doc.mobile
        yield @pencil.playback "voicemail/vm-press"
        yield @pencil.playback "digits/1"

      debug 'Announce mobile query'
      yield @pencil.play 'sms_unknown'
      yield @pencil.playback "voicemail/vm-press"
      yield @pencil.playback "digits/2"

Send via email
--------------

      if @session.doc.email?
        debug 'Announce email'
        yield @pencil.play 'email'
        yield @pencil.spell @session.doc.email
        yield @pencil.playback "voicemail/vm-press"
        yield @pencil.playback "digits/3"

Send via postmail
-----------------

      debug 'Announce snailmail'
      yield @pencil.play 'snailmail'
      yield @pencil.playback "voicemail/vm-press"
      yield @pencil.playback "digits/4"

      yield Promise.delay 10*seconds

      sms_text = """
        Le RIO associé au numéro #{@session.number[0...6]}XXXX est #{@session.rio}.
      """

      clean_number = (number) ->
        number = number.replace /[^\d]/, ''
        number = "33#{number.substr 1}" if number[0] is '0'
        number

      choice = @pencil.clear()
      switch choice
        when '1'
          if @session.doc.mobile?
            number = clean_number @session.doc.mobile
            yield send_sms number, sms_text
              .catch (error) ->
                debug "Send SMS #{error.stack ? error}", number
        when '2'
          yield @pencil.playback 'ivr/ivr-please_enter_the_phone_number'
          yield Promise.delay 16*seconds
          number = clean_number @session.dtmf_buffer
          yield send_sms number, sms_text
            .catch (error) ->
              debug "Send SMS #{error.stack ? error}", number
              yield @pencil.spell 'BP171'
        when '3'
          if @session.doc.email?
            yield send_email @session.doc.email, rios
              .catch (error) ->
                debug "Send email #{error.stack ? error}"
                yield @pencil.spell 'BP177'
        when '4'
          yield send_snailmail @session.doc.nom, @session.doc.adresse_de_facturation, rios
            .catch (error) ->
              debug "Send snail mail #{error.stack ? error}"
              yield @pencil.spell 'BP182'

        else
          debug 'No valid choice was made, aborting.'

      debug 'Thank you.'
      yield @pencil.playback 'ivr/ivr-thank_you_alt'
      yield Promise.delay 3*seconds
      yield @action 'hangup'
