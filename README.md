# COMIDAS
Comidas Carretera
from flask import Flask, request, jsonify
from datetime import datetime
from twilio.rest import Client
import gspread
from oauth2client.service_account import ServiceAccountCredentials

# Configuración de la aplicación Flask
app = Flask(__name__)

# Configuración de Twilio (sustituir con tus propias credenciales)
TWILIO_ACCOUNT_SID = 'YOUR_TWILIO_ACCOUNT_SID'
TWILIO_AUTH_TOKEN = 'YOUR_TWILIO_AUTH_TOKEN'
TWILIO_PHONE_NUMBER = 'YOUR_TWILIO_PHONE_NUMBER'
TWILIO_CLIENT = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)

# Configuración de Google Drive (sustituir con tu propio archivo JSON de credenciales)
scope = ['https://www.googleapis.com/auth/drive']
creds = ServiceAccountCredentials.from_json_keyfile_name('YOUR_GOOGLE_DRIVE_CREDS.json', scope)
client = gspread.authorize(creds)
sheet = client.open('Nombre de tu hoja de Google Drive').sheet1

# Diccionario para almacenar los pedidos por usuario
pedidos_por_usuario = {}

# Ruta para realizar un pedido
@app.route('/pedido', methods=['POST'])
def realizar_pedido():
    usuario = request.json['usuario']
    restaurante = request.json['restaurante']

    # Verificar límite de pedidos por día
    if usuario in pedidos_por_usuario:
        pedidos_realizados_hoy = sum(1 for pedido in pedidos_por_usuario[usuario] if pedido['fecha'] == datetime.now().date())
        if pedidos_realizados_hoy >= 2:
            return jsonify({'message': 'Has alcanzado el límite de pedidos por día'}), 400

    # Registrar el pedido
    pedido = {'usuario': usuario, 'restaurante': restaurante, 'fecha': datetime.now().date()}
    if usuario not in pedidos_por_usuario:
        pedidos_por_usuario[usuario] = [pedido]
    else:
        pedidos_por_usuario[usuario].append(pedido)

    # Enviar mensaje de WhatsApp al restaurante
    mensaje = f'Pedido de {usuario} en {restaurante}'
    TWILIO_CLIENT.messages.create(body=mensaje, from_=TWILIO_PHONE_NUMBER, to='WHATSAPP_NUMBER_OF_RESTAURANT')

    # Alimentar hoja de Google Drive
    fila = [usuario, restaurante, datetime.now().strftime('%Y-%m-%d %H:%M:%S')]
    sheet.append_row(fila)

    return jsonify({'message': 'Pedido realizado correctamente'}), 200

if __name__ == '__main__':
    app.run(debug=True)
