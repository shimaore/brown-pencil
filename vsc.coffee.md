    pkg = require './package'
    @name = "#{pkg.name}:vsc"
    debug = (require 'debug') @name

    assert = require 'assert'
    seem = require 'seem'

    f = (n) -> n.match /^([*#]{1,2})(\d\d)(?:[*](\d+))?(?:[*](\d+))?#?$/
    assert f('*21#')[1] is '*'
    assert f('#21#')[1] is '#'
    assert f('*#21#')[1] is '*#'
    assert f('**21#')[1] is '**'
    assert f('*21#')[2] is '21'
    assert f('*21*35#')[2] is '21'
    assert f('*21*35*89#')[2] is '21'
    assert f('*21*35#')[3] is '35'
    assert f('*21*35*89#')[4] is '89'

    @include = seem ->
      return unless @session.direction is 'egress'

      debug 'Ready'

      d = f @destination
      return unless d

      debug 'Matched', @destination

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
        seem (doc) ->
          yield @pencil.play what # 'Le renvoi sur occupation est'
          if doc["#{what}_enabled"]
            yield @pencil.play 'on' # 'activé'
            yield @pencil.play 'towards' # 'vers le numéro'
            yield @pencil.spell doc["#{what}_number"]
          else
            yield @pencil.play 'off' # 'désactivé'

      flip = (what) ->
        activate: (doc) -> doc[what] = true
        cancel: (doc) -> doc[what] = false
        toggle: (doc) -> doc[what] = not doc[what]
        query: seem (doc) ->
          yield @pencil.play what
          yield @pencil.play if doc[what] then 'on' else 'off'

      target = switch d[2]

        when '69' # CFB
          activate: (doc) ->
            if d[3]?
              doc.cfb_number = d[3]
            doc.cfb_enabled = true
          cancel: (doc) -> doc.cfb_enabled = false
          toggle: (doc) -> doc.cfb_enabled = not doc.cfb_enabled
          query: query_number 'cfb'

* doc.local_number.inv_timer (integer) Time before CFNR (No Answer) forwarding is applied.

        when '61' # CFNR/CFDA
          activate: (doc) ->
            if d[4]?
              doc.inv_timer = parseInt d[4]
            if d[3]?
              doc.cfnr_number = doc.cfda_number = d[3]
            doc.cfnr_enabled = doc.cfda_enabled = true
          cancel: (doc) -> doc.cfnr_enabled = doc.cfda_enabled = false
          toggle: (doc) -> doc.cfnr_enabled = doc.cfda_enabled = not doc.cfnr_enabled
          query: seem (doc) ->
            yield query_number('cfnr').call this, doc
            if doc.cfnr_enabled
              yield @pencil.play 'after'
              yield @action 'phrase', "say-number:#{doc.inv_timer}"
              yield @pencil.play 'seconds'

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

      debug 'Found', {action,target}

      unless action? and target?
        # FIXME play a message indicating invalid choice
        yield @action 'hangup'
        return

Assume we are in a `huge-play` client/egress context, and @session.number contains the document.

      doc = @session.number

Make the change

      unless action is 'query'
        debug 'Calling', {action,doc}
        target[action].call this, doc

Save the change

        debug 'Saving', {action,doc}
        yield @cfg.master_push doc

Announce the new state

      yield @action 'answer'
      debug 'Querying', {doc}
      yield target.query.call this, doc

      debug 'Hangup'
      yield @action 'hangup'
      return
