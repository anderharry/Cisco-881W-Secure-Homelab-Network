# Cisco-881W-Secure-Homelab-Network

Este repositorio documenta la configuración paso a paso de un punto de acceso/router **Cisco-881W** para crear un entorno de red doméstico con seguridad nivel empresarial.

El objetivo principal de este proyecto ha sido segmentar la red Wi-Fi aislando completamente los dispositivos IoT (cámaras, bombillas, Alexa,...) y crear una red de invitados segura, mitigando posibles ataques internos como el *ARP Spoofing*.

## Arquitectura de la Red

Se han configurado dos redes Wi-Fi (SSID) emitiendo desde la misma antena física("DotRadio0"), puenteadas al router principal de la casa que provee la salida a Internet.

  1.- Red IoT (Red_Segura_IoT - VLAN 1) ; Ocultar: el SSID no se emite públicamente (no guest-mode) ; Filtrado: Solo se permite el acceso a dispositivos de confianza (access-list "NÚMERO DE LA LISTA" permit "MAC") ; Cifrado:   WPA2 (aes-ccm).

  2.- Red Invitados (wi-fi_Invitados - VLAN 2) ; Pública: red visible (guest-mode) ; Aislada: los clientes no se pueden comunicar entre sí o con la red local (port-protected, bridge-group) para prevenir posibles escaneos de    red ; Cifrado: WPA2 (aes-ccm).

## Código de Configuración Final 

Aquí dejo el bloque de comandos para aplicar la configuración desde cero en la consola Cisco:
'''bash
configure terminal
dot11 mbssid (sirve para habilitar la emisión de múltiples SSID)

dot11 ssid Red_Segura_IoT (configuración OCULTA de la Red IoT)
  vlan 1
  authentication open
  authentication key-management wpa version 2
  wpa-psk ascii 0 "CONTRASEÑA"
  no guest-mode 
  exit

dot11 ssid wi-fi_Invitados ( configuración VISIBLE de la Red Invitados)
  vlan 2
  authentication key-management wpa version 2
  wpa-psk ascii 0 "CONTRASEÑA"
  guest-mode 
  exit

interface Dot11Radio0 (configuración de la Antena Física)
  encryption vlan 1 mode ciphers aes-ccm
  encryption vlan 2 mode ciphers aes-ccm
  ssid Red_Segura_IoT
  ssid wi-fi_Invitados
  mbssid
  no shutdown
  exit

interface Dot11Radio0.2 (configuración el puente y el aislamiento para invitados)
  encapsulation dot1Q 2
  bridge-group 1
  bridge-group 1 port-protected
  exit
end
write

## Dificultades y soluciones

Durante la configuración en el sistema Cisco IOS, me he encontrado con varios retos interesantes:
  1.- La antena Wi-Fi:
    - PROBLEMAS: Tras configurar el primer SSID, la interfaz Dot11Radio0 se quedaba en estado apagado y lanzaba errores de consola.
    - CAUSA: En las versiones de Cisco IOS para estos equipos, es obligatorio que cada SSID tenga asignada una VLAN para que la radio pueda procesar el tráfico.
    - SOLUCIÓN: Asignar explicitamente vlan 1 a la red IoT y vlan 2 a la red Invitados dentro de la configuración del SSID.
  2.- Móviles atrapados en "Buscando..." (intentando obtener la dirección IP)
    - PROBLEMAS: Al crear la red de invitados e intentar conectar un dispositivo, este se quedaba en un bucle infinito intentando conectarse.
    - CAUSA: Aunque creé la tubería (VLAN 2) para la red de Invitados, no le había indicado a la antena qué traductor de cifrado usar para esa VLAN especifica. El dispositivo intentaba negociar en WPA2, pero la antena no         lo entendía.
    - SOLUCIÓN: Aplicar el cifrado AES a la VLAN 2 directamente dentro de la interfaz física con el comando: encryption vlan 2 mode ciphers aes-ccm.
