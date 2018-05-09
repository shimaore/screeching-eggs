    Call = require '../call'
    {createServer} = require 'net'
    FS = require 'esl'

    port = Math.floor 48000+16000*Math.random()

    describe 'Monitor', ->
      monitor = require '../monitor'

      it 'should connect', (done) ->
        server = createServer (c) ->
          c.setEncoding 'utf8'
          c.on 'end', -> server.close()
          c.on 'data', (data) ->
            if data.match /^auth/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
              return
            if data.match /^event json.*DTMF/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
              wrapper.end()
              done()
              return
            if data.match /^event json/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
              return

          c.write 'Content-Type: auth/request\n\n'
        server.listen ++port
        wrapper = FS.createClient host:'127.0.0.1', port:port
        monitor wrapper, null
        return

      it 'should activate the call', (done) ->
        server = createServer (c) ->
          c.setEncoding 'utf8'
          c.on 'end', -> server.close()
          c.on 'data', (data) ->
            if data.match /^auth/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
              return
            if data.match /^event json/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
            if data.match /DTMF/
              event = JSON.stringify
                'Event-Name': 'CHANNEL_BRIDGE'
                'Bridge-A-Unique-ID': '123'
                'Bridge-B-Unique-ID': '456'
                variable_transfer_disposition: 'fake'
              trigger = ->
                c.write "Content-Type: text/event-json\nContent-Length: #{event.length}\n\n#{event}\n\n\n\n"
              setTimeout trigger, 100
              return

          c.on 'error', console.error

          c.write 'Content-Type: auth/request\n\n'
        server.listen ++port
        wrapper = FS.createClient host:'127.0.0.1', port:port
        Call::get_local_agent = -> Promise.resolve "agent-for-#{@key}"
        Call::set_local_agent = -> Promise.resolve()
        Call::set_remote_agent = -> Promise.resolve()
        Call::transition = -> Promise.resolve true
        Call::interface =
          add: -> Promise.resolve()
        Call::Agent = class Agent
          constructor: (@key) ->
          on_bridge: (call,disposition) ->
            if @key is 'agent-for-123' and call.key is '456' and disposition is 'fake'
              wrapper.end()
              done()
            Promise.resolve()
        monitor wrapper, Call
        return

      it 'should deactivate the call', (done) ->
        server = createServer (c) ->
          c.setEncoding 'utf8'
          c.on 'end', -> server.close()
          c.on 'data', (data) ->
            if data.match /^auth/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
              return
            if data.match /^event json/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
            if data.match /DTMF/
              event = JSON.stringify
                'Event-Name': 'CHANNEL_UNBRIDGE'
                'Bridge-A-Unique-ID': '123'
                'Bridge-B-Unique-ID': '456'
                variable_transfer_disposition: 'fake'
              trigger = ->
                c.write "Content-Type: text/event-json\nContent-Length: #{event.length}\n\n#{event}\n\n\n\n"
              setTimeout trigger, 100
              return

          c.on 'error', console.error

          c.write 'Content-Type: auth/request\n\n'
        server.listen ++port
        wrapper = FS.createClient host:'127.0.0.1', port:port
        Call::get_local_agent = -> Promise.resolve "agent-for-#{@key}"
        Call::set_local_agent = -> Promise.resolve()
        Call::set_remote_agent = -> Promise.resolve()
        Call::transition = -> Promise.resolve true
        Call::interface =
          add: -> Promise.resolve()
        Call::Agent = class Agent
          constructor: (@key) ->
          on_unbridge: (call,disposition) ->
            if @key is 'agent-for-123' and call.key is '456' and disposition is 'fake'
              wrapper.end()
              done()
            Promise.resolve()
        monitor wrapper, Call
        return

      it 'should clear the call', (done) ->
        server = createServer (c) ->
          c.setEncoding 'utf8'
          c.on 'end', -> server.close()
          c.on 'data', (data) ->
            if data.match /^auth/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
              return
            if data.match /^event json/
              c.write 'Content-Type: command/reply\nReply-Text: +OK\n\n'
            if data.match /DTMF/
              event = JSON.stringify
                'Event-Name': 'CHANNEL_HANGUP_COMPLETE'
                'Unique-ID': '123'
                variable_transfer_disposition: 'fake'
              trigger = ->
                c.write "Content-Type: text/event-json\nContent-Length: #{event.length}\n\n#{event}\n\n\n\n"
              setTimeout trigger, 100
              return

          c.on 'error', console.error

          c.write 'Content-Type: auth/request\n\n'
        server.listen ++port
        wrapper = FS.createClient host:'127.0.0.1', port:port
        Call::get_local_agent = -> Promise.resolve "agent-for-#{@key}"
        Call::get_remote_agent = -> Promise.resolve "agent-for-r#{@key}"
        Call::set_local_agent = -> Promise.resolve()
        Call::set_remote_agent = -> Promise.resolve()
        Call::transition = -> Promise.resolve true
        Call::interface =
          add: -> Promise.resolve()
        Call::Agent = class Agent
          constructor: (@key) ->
          on_hangup: (call,disposition) ->
            if @key is 'agent-for-123' and call.key is '123' and disposition is 'fake'
              wrapper.end()
              done()
            Promise.resolve()
        monitor wrapper, Call
        return
