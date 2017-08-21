    pkg = require './package'
    @name = "#{pkg.name}:setup"
    seem = require 'seem'
    path = require 'path'

    @include = ->

      @debug 'Start'

      init_done = false
      init = seem =>
        return if init_done
        yield @set language: 'fr-FR'
        init_done = true

`provisioning` is a `nimble-direction` convention.

      @pencil =
        playback: seem (file) =>
          yield init()
          sound_dir = @cfg.sound_dir ? '/opt/freeswitch/share/freeswitch/sounds'
          sound_path = @cfg.sound_path ? path.join sound_dir, 'fr', 'fr', 'sibylle'
          yield @dtmf.playback path.join sound_path, "#{file}.wav"

        play: seem (file) =>
          yield init()
          yield @dtmf.playback "#{@cfg.provisioning}/config%3Avoice_prompts/#{file}.wav"

        spell: seem (text) =>
          yield init()
          yield @dtmf.phrase "spell,#{text}"

      return
