# Edge WebSocket Tutorial

## Requirements
1. You should know about edge framework, its features and concepts such as:
    1. What is `edge`
    2. What is `context`
    3. What are `actions`

    If you're not familiar with the concepts above — no worries at all 🙂

    it is an open-source library written in a simple and approachable way.
    You can check out the code and read through it at your own pace here:

    📎 [BasisCore.Server.Edge](https://github.com/Manzoomeh/BasisCore.Server.Edge.git)

2. You need to have some knowledge about http sockets

## Lets jump into code😉

### 1. Simple code to get sockets

*Install requirements libraries*
```bash
pip install bclib
```

*Create a `main.py`*
```python
from bclib import edge

options = {
    "endpoint": "<server_ip>:<server_port>",
    "log_error": True  # To see errors occured in app <default ture>
    "log_request": True # To see requests <default true>
}

# Create edge app
app = edge.from_options(
    options=options
)

# Create a router for coming socket messages
@app.socket_action()
def new_socket_message(context: edge.SocketContext):
    # Do something
    print(f"New message received from sessionId={context.message.session_id}")
```

*Run the code*
```bash
python main.py
```

### 2. Create a chat room by edge webSocket

*Install requirements*

```bash
pip install bclib
```

*🧱 Project Structure*

```bash
├── client.py     
├── room.py
├── main.py         
```
*Codes*

*`client.py`*
```python
from bclib import edge
import xml.etree.ElementTree as elem_tree

class Client:

    def __init__(self, session_id: str, cms: edge.DictEx):
        self.cms = cms
        self.session_id = session_id
        self.user_name = None

    async def update_async(self, message: edge.DictEx):
        command = elem_tree.fromstring(message.command)
        if self.user_name:
            user_message = command.get("message")
            if user_message == 'end':
                self.close_async(
                    from_server=True
                )
            else:
                print(f'[{self.user_name}]: {user_message}')
                await ChatRoom.send_to_all_message_async(
                    self.user_name, user_message, 'user')
        else:
            self.user_name = command.get('user-name')
            if self.user_name == ".":
                self.close_async(
                    from_server=True
                )
            else:
                await ChatRoom.send_to_all_message_async(
                    None, f'{self.user_name} Connected!', 'system')
                print(f'{self.user_name} with id {self.session_id} connected!')

    async def close_async(self, from_server: bool):
        if from_server:
            await app.send_message_async(edge.Message.create_disconnect(self.session_id))

        await ChatRoom.send_to_all_message_async(
            None, f"{self.user_name} Disconnected!", 'system')
        print(f'{self.user_name} Disconnected')
```

*`room.py`*
```python
import json
import datetime
from client import Client
from bclib import edge

class ChatRoom:

    __sessions: 'dict[str, Client]' = {}

    @staticmethod
    async def send_to_all_message_async(owner: str, message: str, msg_category: str):
        message_time = datetime.datetime.now().strftime('%H:%M:%S')
        data = {
            "_": {
                "Replace": False
            },
            "data": [
                ["Type", "Owner", "Time", "Message"],
                [msg_category, owner, message_time, message]
            ]
        }
        msg = json.dumps(data, ensure_ascii=True)
        for session_id, _ in ChatRoom.__sessions.items():
            await app.send_message_async(
                edge.Message.create_from_text(session_id, msg))

    @staticmethod
    async def process_message_async(message: edge.Message, cms: edge.DictEx, body: edge.DictEx):
        if(message.type == edge.MessageType.CONNECT):
            ChatRoom.__sessions[message.session_id] = Client(
                message.session_id, cms)
        elif message.type == edge.MessageType.DISCONNECT:
            if message.session_id in ChatRoom.__sessions:
                await ChatRoom.__sessions[message.session_id].close_async(
                    from_server=False
                )
                del ChatRoom.__sessions[message.session_id]
        elif message.type == edge.MessageType.MESSAGE:
            if message.session_id in ChatRoom.__sessions:
                await ChatRoom.__sessions[message.session_id].update_async(
                    message=body
                )
        elif message.type == edge.MessageType.NOT_EXIST:
            if message.session_id in ChatRoom.__sessions:
                del ChatRoom.__sessions[message.session_id]
        elif message.type == edge.MessageType.AD_HOC:
            if message.session_id in ChatRoom.__sessions:
                print("adhoc message receive!")
```

*`main.py`*
```python
from bclib import edge
from room import ChatRoom

options = {
    "endpoint": "<server_ip>:<server_port>",
}

app = edge.from_options(options)

@app.socket_action(
    app.equal("context.message_type", edge.MessageType.NOT_EXIST)
)
async def process_not_exist_message_async(context: edge.SocketContext):
    print("process_not_exist_message")
    await ChatRoom.process_message_async(context.message, None, None)

@ app.socket_action()
async def process_all_other_message_async(context: edge.SocketContext):
    print("process_all_other_message")
    await ChatRoom.process_message_async(context.message,
                                         context.cms, context.body)

```

*Run the code*
```bash
python main.py
```

## 📄 License
MIT

## 👤 Author
Made with ❤️ by \[ManzoomehNegaran]
