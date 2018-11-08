    pkg = require './package'
    @name = "#{pkg.name}:setup"
    {debug} = (require 'tangible') @name
    path = require 'path'

    @include = ->

      debug 'Start'

      init_done = false
      init = =>
        return if init_done
        await @set language: 'fr-FR'
        init_done = true

`provisioning` is a `nimble-direction` convention.

      @pencil =
        playback: (file) =>
          await init()
          sound_dir = @cfg.sound_dir ? '/opt/freeswitch/share/freeswitch/sounds'
          sound_path = @cfg.sound_path ? path.join sound_dir, 'fr', 'fr', 'sibylle'
          await @dtmf.playback path.join sound_path, "#{file}.wav"

        play: (file) =>
          await init()
          await @dtmf.playback "#{@cfg.provisioning}/config%3Avoice_prompts/#{file}.wav"

        spell: (text) =>
          await init()
          await @dtmf.phrase "spell,#{text}"

      return
