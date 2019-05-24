    pkg = require './package'
    @name = "#{pkg.name}:vsc"
    {debug} = (require 'tangible') @name
    Nimble = require 'nimble-direction'

    seconds = 1000
    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    assert = require 'assert'

    f = (n) -> n.match /^([*#]{1,2})(\d{2,3})(?:[*](\d+))?(?:[*](\d+))?#?$/
    assert f('*21#')[1] is '*'
    assert f('#21#')[1] is '#'
    assert f('*#21#')[1] is '*#'
    assert f('**21#')[1] is '**'
    assert f('*21#')[2] is '21'
    assert f('*21#')[3] is undefined
    assert f('*21*35#')[2] is '21'
    assert f('*21*35*89#')[2] is '21'
    assert f('*21*35#')[3] is '35'
    assert f('*21*35#')[4] is undefined
    assert f('*21*35*89#')[4] is '89'

References
----------

- ETSI/HF: https://portal.etsi.org/TBSiteMap/HF/hfservicecodes.aspx
- MMI procedures of https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=570 , code in Annex B (themselves based on ETSI/HF and ITU-T E.131)
- https://en.wikipedia.org/wiki/Call_forwarding#Europe

    @include = ->
      return unless @session?.direction is 'egress'
      return if @session.forwarding is true

      debug 'Ready'

      {master_push} = Nimble @cfg

      {VOICEMAIL} = @session

Make sure we don't match (in case @session.VOICEMAIL was not defined properly).

      VOICEMAIL ?= 'vm'

      d = f @destination
      return unless d

      debug 'Matched', @destination

      @direction 'vsc'

      action = switch d[1]
        when '*'
          'activate'
        when '#'
          'cancel'
        when '*#'
          'query'
        when '**'
          'toggle' # 3GPP 'registration'

3GPP also supports '##' for 'erasure'

      query_number = (what) ->
        (doc) ->
          await @pencil.play what # 'Le renvoi sur occupation est'
          if doc["#{what}_enabled"]
            await @pencil.play 'on' # 'activé'
            if doc["#{what}_voicemail"]
              await @pencil.play 'towards-voicemail' # 'vers la messagerie'
            else
              await @pencil.play 'towards' # 'vers le numéro'
              await @pencil.spell doc["#{what}_number"]
          else
            await @pencil.play 'off' # 'désactivé'

      flip = (what) ->
        activate: (doc) -> doc[what] = true
        cancel: (doc) -> doc[what] = false
        toggle: (doc) -> doc[what] = not doc[what]
        query: (doc) ->
          await @pencil.play what
          await @pencil.play if doc[what] then 'on' else 'off'

      target = switch d[2]

CF Busy is 67 for 3GPP and ETSI

        when '69', '67' # CFB
          activate: (doc) ->
            if d[3]?
              if d[3] is VOICEMAIL
                doc.cfb_voicemail = true
              else
                doc.cfb_voicemail = false
                doc.cfb_number = d[3]
            doc.cfb_enabled = true
          cancel: (doc) -> doc.cfb_enabled = false
          toggle: (doc) -> doc.cfb_enabled = not doc.cfb_enabled
          query: query_number 'cfb'

* doc.local_number.inv_timer (integer) Time before CFNR (No Answer) forwarding is applied.

ETSI, 3GPP use 61 for CFDA (No Reply), 62 for CFNR (Not Reachable)

        when '61', '62' # CFDA/CFNR
          activate: (doc) ->
            if d[4]?
              doc.inv_timer = parseInt d[4]
            if d[3]?
              if d[3] is VOICEMAIL
                doc.cfnr_voicemail = doc.cfda_voicemail = true
              else
                doc.cfnr_voicemail = doc.cfda_voicemail = false
                doc.cfnr_number = doc.cfda_number = d[3]
            doc.cfnr_enabled = doc.cfda_enabled = true
          cancel: (doc) -> doc.cfnr_enabled = doc.cfda_enabled = false
          toggle: (doc) -> doc.cfnr_enabled = doc.cfda_enabled = not doc.cfnr_enabled
          query: (doc) ->
            await query_number('cfnr').call this, doc
            if doc.cfnr_enabled
              await @pencil.play 'after'
              await @action 'phrase', "say-number:#{doc.inv_timer}"
              await @pencil.play 'seconds'

        when '610' # CFDA only
          activate: (doc) ->
            if d[4]?
              doc.inv_timer = parseInt d[4]
            if d[3]?
              if d[3] is VOICEMAIL
                doc.cfda_voicemail = true
              else
                doc.cfda_voicemail = false
                doc.cfda_number = d[3]
            doc.cfda_enabled = true
          cancel: (doc) -> doc.cfda_enabled = false
          toggle: (doc) -> doc.cfda_enabled = not doc.cfda_enabled
          query: (doc) ->
            await query_number('cfda').call this, doc
            if doc.cfda_enabled
              await @pencil.play 'after'
              await @action 'phrase', "say-number:#{doc.inv_timer}"
              await @pencil.play 'seconds'

        when '620' # CFNR only
          activate: (doc) ->
            if d[4]?
              doc.inv_timer = parseInt d[4]
            if d[3]?
              if d[3] is VOICEMAIL
                doc.cfnr_voicemail = true
              else
                doc.cfnr_voicemail = false
                doc.cfnr_number = d[3]
            doc.cfnr_enabled = true
          cancel: (doc) -> doc.cfnr_enabled = false
          toggle: (doc) -> doc.cfnr_enabled = not doc.cfnr_enabled
          query: (doc) ->
            await query_number('cfnr').call this, doc
            if doc.cfnr_enabled
              await @pencil.play 'after'
              await @action 'phrase', "say-number:#{doc.inv_timer}"
              await @pencil.play 'seconds'

ETSI Call Forwarding, Unconditional to any number; 3GPP CFU

        when '21' # CFA
          activate: (doc) ->
            if d[3]?
              if d[3] is VOICEMAIL
                doc.cfa_voicemail = true
              else
                doc.cfa_voicemail = false
                doc.cfa_number = d[3]
            doc.cfa_enabled = true
          cancel: (doc) -> doc.cfa_enabled = false
          toggle: (doc) -> doc.cfa_enabled = not doc.cfa_enabled
          query: query_number 'cfa'

ETSI: 934 (CLIR) 935 (no CLIP)
NANPA: 77

        when '82', '934', '935' # Reject Anonymous
          flip 'reject_anonymous'

ETSI 31 (CLIR)

        when '31' # Privacy
          flip 'privacy'

ETSI 26 (announcement), 49 (in hunt group)
NANPA: 78 and 79

        when '26' # DND
          flip 'dnd'

      debug 'Found', {action,target}

      unless action? and target?
        # FIXME play a message indicating invalid choice
        await @action 'hangup'
        return

Assume we are in a `huge-play` client/egress context, and @session.number contains the document.

      doc = @session.number

Make the change

      unless action is 'query'
        debug 'Calling', {action,doc}
        target[action].call this, doc

Save the change

        debug 'Saving', {action,doc}
        await master_push doc

Announce the new state

      await @action 'answer'
      debug 'Querying', {doc}
      await target.query.call this, doc

      debug 'Hangup'
      await sleep 3*seconds
      await @action 'hangup'
      return
