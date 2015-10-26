    pkg = require './package'
    @name = "#{pkg.name}:appel_3179"
    debug = (require 'debug') @name
    seem = require 'seem'
    CaringBand = require 'caring-band'
    Promise = require 'bluebird'
    seconds = 1000

    @server_pre = ->
      @cfg.statistics ?= new CaringBand()

    @include = seem (ctx) ->

      debug 'Start',
        direction: @session.direction
        dialplan: @session.dialplan
        country: @session.country
        destination: @destination

`provisioning` is a `nimble-direction` convention.

      ctx.rio ?=
        playback: (file) =>
          @action 'playback', "#{file.wav}"
        play: (file) =>
          @action 'playback', "#{@cfg.provisioning}/config%3Avoice_prompts/#{file}.wav"

        spell: seem (text) =>
          @action 'phrase', "say-iterated,#{text}"

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

      @statistics.emit 'rio-request', source:@source

      @call.on 'DTMF', (res) ->
        @session.dtmf_buffer ?= ''
        @session.dtmf_buffer += res.body['DTMF-Digit']
        debug "dtmf_buffer = `#{@session.dtmf_buffer}`"

      get_rio_index = (rios) ->
        @session.dtmf_number = ''
        if rios.lengh > 1
          yield @rio.play 'welcome_internal'
          yield @rio.play 'enter_number_first'
          for number,rio in rios
            yield @rio.play 'for_number'
            yield @rio.spell number
            yield @rio.playback "voicemail/vm-press"
            yield @rio.playback "digits/#{i+1}"
            yield Promise.delay 3*seconds
        else
          @session.dtmf_number = '1'

        if @session.dtmf_number.length is 0
          get_rio_index()
        else
          index = parseInt(@session.dtmf_number)-1
          @session.number = rios[index].number
          @session.rio = rios[index].rio

      retrieve_user = ->
        Promise.resolve
          rios: {
            '0972222713': 'OO Q RRRRR CCC'
          }
          email: 'stephane@shimaore.net'
          nom: 'Stéphane Test'
          addresse_de_facturation: '42 rue des Glycines\nAppt 42\n99123 Gogrville'

      @session.doc = yield retrieve_user @source
        .catch (error) =>
          @statistics.emit 'rio-failed', source:@source
          @action 'respond', 503
          @end

      assert @session.doc.rios
      rios = ({number,rio} for own number,rio of @session.doc.rios)
      index = yield get_rio_index rios
      yield @rio.play 'the_rio'
      yield @rio.spell @session.number
      yield @rio.play 'is'
      yield @rio.spell @session.rio
      yield @rio.play 'dont_cancel'

Destination
===========

      @session.dtmf_number = ''

Send via SMS
------------

We don't have a proper list of customer mobile numbers at this time. Skip until we can provide this.

      ###
      if @session.doc.mobile?
        yield @rio.play 'sms_known'
        yield @rio.spell @session.doc.mobile
        yield @rio.playback "voicemail/vm-press"
        yield @rio.playback "digits/1"

      yield @rio.play 'sms_unknown'
      yield @rio.playback "digits/2"
      ###

      send_sms = (recipient,text) =>
        yield @rio.playback 'ivr/ivr-thank_you_alt'

      send_email = (recipient,html) =>
        yield @rio.playback 'ivr/ivr-thank_you_alt'

      send_snailmail = (recipient,address) =>
        yield @rio.playback 'ivr/ivr-thank_you_alt'


Send via email
--------------

      if @session.doc.email?
        yield @rio.play 'email'
        yield @rio.spell @session.doc.email
        yield @rio.playback "voicemail/vm-press"
        yield @rio.playback "digits/3"

Send via postmail
-----------------

      yield @rio.play 'snailmail'
      yield @rio.playback "voicemail/vm-press"
      yield @rio.playback "digits/4"

      yield Promise.delay 10*seconds

      sms_text = """
        Le RIO associé au numéro #{@session.number[0...6]}XXXX est #{@session.rio}.
      """

      switch @session.dtmf_number
        when '1'
          if @session.doc.mobile?
            send_sms @session.doc.mobile, sms_text
        when '2'
          @session.dtmf_number = ''
          yield @rio.playback 'ivr/ivr-please_enter_the_phone_number'
          yield Promise.delay 16*seconds
          send_sms @session.dtmf_number, sms_text
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
