    RedisClient = require 'normal-key/client'

    class Call extends RedisClient

      Agent: null # to be defined

      constructor: (key) ->
        super 'call', key
        @__bridged_key = "#{@class_name}-#{key}-Sb"

      bridged: (call,data) ->
        await @interface.add @__bridged_key, call.key
        await @transition 'bridge', data ? {call}

      unbridged: (call,data) ->
        await @interface.remove @__bridged_key, call.key
        last_call = 0 is await @interface.count @__bridged_key
        if last_call
          debug 'unbridged', @key, call.key
          await @transition 'unbridge', data ? {call}
        else
          debug 'unbridged: ignore', @key, call.key
          false

### `on_bridge`

      on_bridge: (b_call,disposition) ->
        debug 'on_bridge', @key, b_call.key, disposition
        a_call = this

        a_agent_key = await a_call.get_local_agent()
        b_agent_key = await b_call.get_local_agent()

        debug 'on_bridge', @key, a_agent_key, b_call.key, b_agent_key, disposition

        await a_call.bridged b_call, if b_agent_key? then agent_call:b_call else call:b_call
        await b_call.bridged a_call, if a_agent_key? then agent_call:a_call else call:a_call

        await a_call.set_remote_agent b_agent_key
        await b_call.set_remote_agent a_agent_key

        if a_agent_key?
          a_agent = new @Agent a_agent_key
          await a_agent.on_bridge b_call, disposition

        if b_agent_key?
          b_agent = new @Agent b_agent_key
          await b_agent.on_bridge a_call, disposition

        return

### `on_dtmf`

      on_dtmf: (digit) ->
        debug 'on_dtmf', @key, digit
        a_call = this
        a_agent_key = await a_call.get_local_agent()
        if a_agent_key?
          a_agent = new @Agent a_agent_key
          await a_agent.dmtf digit

        return

### `on_unbridge`

      on_unbridge: (b_call,disposition) ->
        debug 'on_unbridge', @key, b_call.key, disposition
        a_call = this

        a_call.unbridged b_call
        b_call.unbridged a_call

        a_agent_key = await a_call.get_local_agent()
        b_agent_key = await b_call.get_local_agent()

        debug 'on_unbridge', @key, a_agent_key, b_call.key, b_agent_key, disposition

If the call was transfered, clear the list of matching calls (used by `unbridge_except`).

        if disposition is 'replaced'
          await a_call.clear()

Remove the other call leg from each agent's list.

        if a_agent_key?
          a_agent = new @Agent a_agent_key
          await a_agent.on_unbridge b_call, disposition

        if b_agent_key?
          b_agent = new @Agent b_agent_key
          await b_agent.on_unbridge a_call, disposition

And remove the link to the remote agent as well.

        ###
        # This might run into issues.
        # We really need an atomic "set to this new value if the value was" operation.
        ###
        await a_call.set_remote_agent null if b_agent_key is await a_call.get_remote_agent()

If the call was transfered,

        if disposition is 'replaced'
          # expect body.variable_endpoint_disposition is 'ATTENDED_TRANSFER'
          await b_call.set_remote_agent a_agent_key
        else
          await b_call.set_remote_agent null if a_agent_key is await b_call.get_remote_agent()

        return

      on_hangup: (disposition) ->
        debug 'on_hangup', @key, disposition
        a_call = this

        a_agent_key = await a_call.get_local_agent()
        b_agent_key = await a_call.get_remote_agent()

        debug 'on_hangup', @key, a_agent_key, b_agent_key, disposition

If the call was transfered, clear the list of matching calls (used by `unbridge_except`).

        if disposition is 'replaced'
          await a_call.clear()

The call leg might be bound to an agent.

        if a_agent_key?
          a_agent = new @Agent a_agent_key
          await a_agent.on_hangup a_call, disposition

The call leg might be connected to an agent.

        if b_agent_key?
          b_agent = new @Agent b_agent_key
          await b_agent.on_hangup a_call, disposition

        await a_call.set_remote_agent null

If the call was bound to an agent, differentiate the handling

        event =
          if a_agent_key?
            switch disposition
              when 'recv_replace', 'bridge'
                'transferred' # 'agent_transfer'
              when 'replaced'
                'transferred'
              else
                'hungup'
          else
            'hangup'

        await a_call.transition event
        return

    module.exports = Call
    {debug} = (require 'tangible') 'screeching-eggs:call'
