# Tchat-Django-Channel

# tuto tchat

# creer un  projet  mysite

# creer une application chat
# installer Django Channels :

				pip install -U channels

# creer un dossier templates dans l'application chat et ensuite un dossier chat dans le dossier templates 

# ensuite dans le dossier templates/chat creer une page  index.html et ajouter ce code:

				<!DOCTYPE html>
				<html>
				<head>
						<meta charset="utf-8"/>
						<title>Chat Rooms</title>
				</head>
				<body>
						What chat room would you like to enter?<br/>
						<input id="room-name-input" type="text" size="100"/><br/>
						<input id="room-name-submit" type="button" value="Enter"/>

						<script>
								document.querySelector('#room-name-input').focus();
								document.querySelector('#room-name-input').onkeyup = function(e) {
										if (e.keyCode === 13) {  // enter, return
												document.querySelector('#room-name-submit').click();
										}
								};

								document.querySelector('#room-name-submit').onclick = function(e) {
										var roomName = document.querySelector('#room-name-input').value;
										window.location.pathname = '/chat/' + roomName + '/';
								};
						</script>
				</body>
				</html>


# creer la vue views.py en ajoutant le code suivant :


				from django.shortcuts import render

				def index(request):
						return render(request, 'chat/index.html', {})



# creer la route urls.py de l'application , ajouter le code suivant :


				from django.urls import path

				from . import views

				urlpatterns = [
						path('', views.index, name='index'),
				]


# creer la route urls.py du projet mysite, et en ajouter le code suivant :


				from django.conf.urls import include
				from django.urls import path
				from django.contrib import admin

				urlpatterns = [
						path('chat/', include('chat.urls')),
						path('admin/', admin.site.urls),
				]

# creer un fichier routing.py dans le projet mysite , et ajouter le code suivant :

				from channels.routing import ProtocolTypeRouter

				application = ProtocolTypeRouter({
						# (http->django views is added by default)
				})


# ajouter channels dans le settings.py comme ceci :

				INSTALLED_APPS = [
						'channels',
						'chat',
						'django.contrib.admin',
						'django.contrib.auth',
						'django.contrib.contenttypes',
						'django.contrib.sessions',
						'django.contrib.messages',
						'django.contrib.staticfiles',
				]

# ensuite ajouter cela aussi apres le INSTALLED_APPS dans le settings.py :

				ASGI_APPLICATION = 'mysite.routing.application'


# creer un dossier chat dans le dossier templates precedent et creer un dossier index.html et ajouter le code suivant :


				<!-- chat/templates/chat/room.html -->
				<!DOCTYPE html>
				<html>
				<head>
						<meta charset="utf-8"/>
						<title>Chat Room</title>
				</head>
				<body>
						<textarea id="chat-log" cols="100" rows="20"></textarea><br/>
						<input id="chat-message-input" type="text" size="100"/><br/>
						<input id="chat-message-submit" type="button" value="Send"/>
				</body>
				<script>
						var roomName = {{ room_name_json }};

						var chatSocket = new WebSocket(
								'ws://' + window.location.host +
								'/ws/chat/' + roomName + '/');

						chatSocket.onmessage = function(e) {
								var data = JSON.parse(e.data);
								var message = data['message'];
								document.querySelector('#chat-log').value += (message + '\n');
						};

						chatSocket.onclose = function(e) {
								console.error('Chat socket closed unexpectedly');
						};

						document.querySelector('#chat-message-input').focus();
						document.querySelector('#chat-message-input').onkeyup = function(e) {
								if (e.keyCode === 13) {  // enter, return
										document.querySelector('#chat-message-submit').click();
								}
						};

						document.querySelector('#chat-message-submit').onclick = function(e) {
								var messageInputDom = document.querySelector('#chat-message-input');
								var message = messageInputDom.value;
								chatSocket.send(JSON.stringify({
										'message': message
								}));

								messageInputDom.value = '';
						};
				</script>
				</html>


# remplacer le code du views.py de lappli actuel par le code suivant : 

				from django.shortcuts import render
				from django.utils.safestring import mark_safe
				import json

				def index(request):
						return render(request, 'chat/index.html', {})

				def room(request, room_name):
						return render(request, 'chat/room.html', {
								'room_name_json': mark_safe(json.dumps(room_name))
						})
		
		
# remplacer le code du urls.py de lappli actuel par le code suivant : 


				from django.urls import path

				from . import views

				urlpatterns = [
						path('', views.index, name='index'),
						path('<str:room_name>/', views.room, name='room'),
				]


# creer un fichier chat/consumers.py et ajouter le code suivant :


				from channels.generic.websocket import WebsocketConsumer
				import json

				class ChatConsumer(WebsocketConsumer):
						def connect(self):
								self.accept()

						def disconnect(self, close_code):
								pass

						def receive(self, text_data):
								text_data_json = json.loads(text_data)
								message = text_data_json['message']

								self.send(text_data=json.dumps({
										'message': message
								}))
								
# remplacer le code du routing.py de lappli actuel par le code suivant : 


			# chat/routing.py
			from django.urls import re_path

			from . import consumers

			websocket_urlpatterns = [
					re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer),
			]


# creer un fichier mysite/routing.py et ajouter le code suivant :


			from channels.auth import AuthMiddlewareStack
			from channels.routing import ProtocolTypeRouter, URLRouter
			import chat.routing

			application = ProtocolTypeRouter({
					# (http->django views is added by default)
					'websocket': AuthMiddlewareStack(
							URLRouter(
									chat.routing.websocket_urlpatterns
							)
					),
			})


# installer redis en executant le code suivant :

			$ pip3 install channels_redis


# ajouter le code suivant au settings.py :


			ASGI_APPLICATION = 'mysite.routing.application'
			CHANNEL_LAYERS = {
					'default': {
							'BACKEND': 'channels_redis.core.RedisChannelLayer',
							'CONFIG': {
									"hosts": [('127.0.0.1', 6379)],
							},
					},
			}
			
# remplacer le code customers.py actuel par le code suivant : 


				# chat/consumers.py
				from asgiref.sync import async_to_sync
				from channels.generic.websocket import WebsocketConsumer
				import json

				class ChatConsumer(WebsocketConsumer):
						def connect(self):
								self.room_name = self.scope['url_route']['kwargs']['room_name']
								self.room_group_name = 'chat_%s' % self.room_name

								# Join room group
								async_to_sync(self.channel_layer.group_add)(
										self.room_group_name,
										self.channel_name
								)

								self.accept()

						def disconnect(self, close_code):
								# Leave room group
								async_to_sync(self.channel_layer.group_discard)(
										self.room_group_name,
										self.channel_name
								)

						# Receive message from WebSocket
						def receive(self, text_data):
								text_data_json = json.loads(text_data)
								message = text_data_json['message']

								# Send message to room group
								async_to_sync(self.channel_layer.group_send)(
										self.room_group_name,
										{
												'type': 'chat_message',
												'message': message
										}
								)

						# Receive message from room group
						def chat_message(self, event):
								message = event['message']

								# Send message to WebSocket
								self.send(text_data=json.dumps({
										'message': message
								}))

