The main goal is to keep track of the agent that might be connected to a call (especially during transfers) and update the state accordingly.

    monitor = (wrapper,Call) ->

      wrapper.on 'connect', (client) ->

        client.on 'CHANNEL_BRIDGE', foot ({body}) ->
          a_uuid = body['Bridge-A-Unique-ID']
          b_uuid = body['Bridge-B-Unique-ID']
          disposition = body?.variable_transfer_disposition

          a_call = new Call a_uuid
          b_call = new Call b_uuid

          await a_call.on_bridge b_call, disposition
          return

        client.on 'CHANNEL_UNBRIDGE', foot ({body}) ->
          a_uuid = body['Bridge-A-Unique-ID']
          b_uuid = body['Bridge-B-Unique-ID']
          disposition = body?.variable_transfer_disposition
          # body.variable_endpoint_disposition
          a_call = new Call a_uuid
          b_call = new Call b_uuid

          await a_call.on_unbridge b_call, disposition
          return

        client.on 'DTMF', foot ({body}) ->
          a_uuid = body['Unique-ID']
          digit = body['DTMF-Digit']

          a_call = new Call a_uuid
          await a_call.dtmf digit
          return

        client.on 'CHANNEL_HANGUP_COMPLETE', foot ({body}) ->
          a_uuid = body['Unique-ID']
          disposition = body?.variable_transfer_disposition
          # body.variable_endpoint_disposition

In case we get `UNBRIDGE` and `CHANNEL_COMPLETE` at the same time, let `UNBRIDGE` transition first.

          await nextTick()

          a_call = new Call a_uuid
          await a_call.on_hangup disposition
          return

        client.event_json 'CHANNEL_HANGUP_COMPLETE', 'DTMF', 'CHANNEL_BRIDGE', 'CHANNEL_UNBRIDGE'
        return

    module.exports = monitor
    nextTick = -> new Promise (resolve) -> process.nextTick resolve
    {foot} = (require 'tangible') 'screeching-eggs:monitor'
