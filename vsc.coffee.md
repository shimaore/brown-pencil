    pkg = require './package'
    @name = "#{pkg.name}:vsc"

    assert = require 'assert'
    seem = require 'seem'

    f = (n) -> n.match /^([*#]{1,2})(\d\d)(?:[*](\d+))?(?:[*](\d+))?#$/
    assert f('*21#')[1] is '*'
    assert f('*21#')[2] is '21'
    assert f('*21*35#')[2] is '21'
    assert f('*21*35*89#')[2] is '21'
    assert f('*21*35#')[3] is '35'
    assert f('*21*35*89#')[4] is '89'

    @include = seem ->
      return unless @session.direction is 'egress'

      d = f @destination
      return unless d

      action = switch d[1]
        when '*'
          'activate'
        when '#'
          'cancel'
        when '*#'
          'query'
        when '**'
          'toggle'

      query_number = (what) ->
        seem (doc,what) ->
          yield @pencil.playback what # 'Le renvoi sur occupation est'
          if doc["#{what}_enabled"]
            yield @pencil.playback 'on' # 'activé'
            yield @pencil.playback 'towards' # 'vers le numéro'
            yield @pencil.spell doc["#{what}_number"]
          else
            yield @pencil.playback 'off' # 'désactivé'

      flip = (what) ->
        activate: (doc) -> doc[what] = true
        cancel: (doc) -> doc[what] = false
        toggle: (doc) -> doc[what] = not doc[what]
        query: seem (doc) ->
          yield @pencil.playback what
          yield @pencil.playback if doc[what] then 'on' else 'off'

      target = switch d[2]

        when '69' # CFB
          activate: (doc) ->
            if d[3]?
              doc.cfb_number = d[3]
            doc.cfb_enabled = true
          cancel: (doc) -> doc.cfb_enabled = false
          toggle: (doc) -> doc.cfb_enabled = not doc.cfb_enabled
          query: query_number 'cfb'

        when '61' # CFNR
          activate: (doc) ->
            if d[4]?
              doc.inv_timer = parseInt d[4]
            if d[3]?
              doc.cfnr_number = d[3]
            doc.cfnr_enabled = true
          cancel: (doc) -> doc.cfnr_enabled = false
          toggle: (doc) -> doc.cfnr_enabled = not doc.cfnr_enabled
          query: seem (doc) ->
            yield query_number('cfnr') doc
            if doc.cfnr_enabled
              yield @pencil.playback 'after'
              yield @action 'phrase', "say:#{doc.inv_time}"
              yield @pencil.playback 'seconds'

        when '21' # CFA
          activate: (doc) ->
            if d[3]?
              doc.cfa_number = d[3]
            doc.cfa_enabled = true
          cancel: (doc) -> doc.cfa_enabled = false
          toggle: (doc) -> doc.cfa_enabled = not doc.cfa_enabled
          query: query_number 'cfa'

        when '82' # Reject Anonymous
          flip 'reject_anonymous'

        when '31' # Privacy
          flip 'privacy'

      unless action? and target?
        # FIXME play a message indicating invalid choice
        @action 'hangup'
        return

Assume we are in a `huge-play` client/egress context, and @session.number contains the document.

      doc = @session.number

Make the change

      unless action is 'query'
        target[action].call this, doc

Save the change

        yield @cfg.prov.put doc

Announce the new state

      target.query.call this, doc

      @action 'hangup'
