    pkg = require './package'
    @name = "#{pkg.name}:appel_3179"
    {debug} = (require 'tangible') @name
    seconds = 1000

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    bucket = try require 'glorious-bucket'

    @include = ->

      unless bucket?
        debug.dev 'Missing glorious-bucket'
        return

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

      return unless @session?.direction is 'egress'
      return unless @session.dialplan is 'national'
      return unless @session.country is 'fr'
      return unless @destination is '3179'

      return if @session.forwarding is true

      debug 'Start',
        direction: @session.direction
        dialplan: @session.dialplan
        country: @session.country
        destination: @destination

      {send_sms,send_email,send_snailmail,retrieve_user} = bucket @cfg

Prevent further processing.

      @direction 'rio-request'

      @report state:'rio-request', source:@source

      await @action 'answer'
      await @pencil.play 'welcome_internal'

      get_rio_index = (rios) =>
        debug "get_rio_index", {rios}
        @dtmf.clear()
        debug 'get_rio_index length', rios.length
        if rios.length > 1
          debug 'Listing RIOs'
          await @pencil.play 'enter_number_first'
          for v,i in rios when v.rio?
            debug 'get_rio_index record', i,v
            await @pencil.play 'for_number'
            await @pencil.spell v.number
            await @pencil.playback "voicemail/vm-press"
            await @pencil.playback "digits/#{i+1}"
          buffer = await @dtmf.expect 1, rios.length.toString().length
        else
          debug 'Only one RIO available'
          buffer = '1'

        if buffer.length is 0
          await get_rio_index rios
        else
          debug 'User selection', buffer
          index = parseInt(buffer)-1
          @session.rio_number = rios[index].number
          @session.rio = rios[index].rio
        return

      debug 'retrieve user', @source

      @session.doc = await retrieve_user(@source)
        .catch (error) =>
          debug.dev "retrieve user: #{error.stack ? error}"
          @report state:'rio-failed', source:@source
          await @action 'respond', 503
          @direction 'rio-failed'
          null

      debug 'retrieve user returned', @session.doc
      return unless @session.doc?

      unless @session.doc.rios?
        debug 'No RIOs'
        await @pencil.spell 'BP93'
        await @action 'hangup'
        return

      rios = ({number,rio} for own number,rio of @session.doc.rios)
      await get_rio_index rios
        .catch (error) ->
          debug.dev "get_rio_index failed #{error.stack ? error}"
          get_rio_index rios

      await @pencil.play 'the_rio'
      await @pencil.spell @session.rio_number
      await @pencil.play 'is'
      await @pencil.spell @session.rio
      await @pencil.play 'dont_cancel'


Destination
===========

      @dtmf.clear()

Send via SMS
------------

      if @session.doc.mobile?
        debug 'Announce mobile', @session.doc.mobile
        await @pencil.play 'sms_known'
        await @pencil.spell @session.doc.mobile
        await @pencil.playback "voicemail/vm-press"
        await @pencil.playback "digits/1"

      debug 'Announce mobile query'
      await @pencil.play 'sms_unknown'
      await @pencil.playback "voicemail/vm-press"
      await @pencil.playback "digits/2"

Send via email
--------------

      if @session.doc.email?
        debug 'Announce email'
        await @pencil.play 'email'
        await @pencil.spell @session.doc.email
        await @pencil.playback "voicemail/vm-press"
        await @pencil.playback "digits/3"

Send via postmail
-----------------

      debug 'Announce snailmail'
      await @pencil.play 'snailmail'
      await @pencil.playback "voicemail/vm-press"
      await @pencil.playback "digits/4"

      sms_text = """
        Le RIO associé au numéro #{@session.rio_number[0...6]}XXXX est #{@session.rio}.
      """

      clean_number = (number) ->
        number = number.replace /[^\d]/, ''
        number = "33#{number.substr 1}" if number[0] is '0'
        number

      await @pencil.playback "silence_stream://10000"

      switch await @dtmf.expect 1, 1
        when '1'
          if @session.doc.mobile?
            number = clean_number @session.doc.mobile
            await send_sms number, sms_text
              .catch (error) =>
                debug.dev "Send SMS #{error.stack ? error}", number
                await @pencil.spell 'BP167'
        when '2'
          @dtmf.clear()
          await @pencil.playback 'ivr/ivr-please_enter_the_phone_number'
          number = clean_number await @dtmf.expect 10, 10
          await send_sms number, sms_text
            .catch (error) =>
              debug "Send SMS #{error.stack ? error}", number
              await @pencil.spell 'BP171'
        when '3'
          if @session.doc.email?
            await send_email @session.doc.email, rios
              .catch (error) =>
                debug "Send email #{error.stack ? error}"
                await @pencil.spell 'BP177'
        when '4'
          await send_snailmail @session.doc.nom, @session.doc.adresse_de_facturation, rios
            .catch (error) =>
              debug "Send snail mail #{error.stack ? error}"
              await @pencil.spell 'BP182'

        else
          debug 'No valid choice was made, aborting.'

      debug 'Thank you.'
      await @pencil.playback 'ivr/ivr-thank_you_alt'
      await sleep 3*seconds
      await @action 'hangup'
