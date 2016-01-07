    pkg = require './package'
    @name = "#{pkg.name}:setup"
    debug = (require 'debug') @name
    seem = require 'seem'
    CaringBand = require 'caring-band'
    path = require 'path'

    @server_pre = ->
      @cfg.statistics ?= new CaringBand()

    @include = (ctx) ->

      debug 'Start',
        direction: @session.direction
        dialplan: @session.dialplan
        country: @session.country
        destination: @destination

      @set
        language: 'fr-FR'
        playback_terminators: '1234567890#*'

`provisioning` is a `nimble-direction` convention.

      ctx.pencil ?=
        playback: (file) =>
          return Promise.resolve() if enough()
          sound_dir = @cfg.sound_dir ? '/opt/freeswitch/share/freeswitch/sounds'
          sound_path = @cfg.sound_path ? path.join sound_dir, 'fr', 'fr', 'sibylle'
          @action 'playback', path.join sound_path, "#{file}.wav"
        play: (file) =>
          return Promise.resolve() if enough()
          @action 'playback', "#{@cfg.provisioning}/config%3Avoice_prompts/#{file}.wav"

        spell: (text) =>
          return Promise.resolve() if enough()
          @action 'phrase', "spell,#{text}"

        clear: (n) =>
          if n?
            @session.dtmf_min_length = n
          r = @session.dtmf_buffer
          @session.dtmf_buffer = ''
          r

      ctx.pencil.clear 1
      enough = @session.enough = =>
        @session.dtmf_buffer.length >= @session.dtmf_min_length

      @call.on 'DTMF', (res) =>
        @session.dtmf_buffer ?= ''
        @session.dtmf_buffer += res.body['DTMF-Digit']
        debug "dtmf_buffer = `#{@session.dtmf_buffer}`"

      return
