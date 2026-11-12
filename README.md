# IOT-agent-home-assistant

Home Assistant, Mosquitto e Caddy em contêineres, para a atividade de IoT.

## Preparação na VPS

1. Aponte um registro DNS do domínio para o IP público da VPS. Ajuste ou remova qualquer registro AAAA que não corresponda ao IPv6 da VPS.
2. Substitua `casa.seudominio.com` em `Caddyfile` pelo domínio escolhido. Libere as portas TCP 80 e 443 no provedor e no firewall da VPS; elas precisam estar disponíveis para o Caddy.
3. Confira se a rede `172.28.50.0/24` definida no Compose não conflita com outras redes da VPS.
4. No diretório do projeto, crie o usuário MQTT e sua senha interativamente:

   ```bash
   docker run --rm -it \
     -v "$PWD/mosquitto/config:/mosquitto/config" \
     --entrypoint mosquitto_passwd \
     eclipse-mosquitto:2 \
     -c /mosquitto/config/passwd homeassistant
   chmod 644 mosquitto/config/passwd
   ```

   O arquivo contém hashes de senha e não deve ser versionado. O parâmetro `-c` sobrescreve um arquivo existente; para adicionar outro usuário depois, omita-o e reinicie o Mosquitto.

5. Inicie os serviços:

   ```bash
   docker compose up -d
   docker compose logs -f caddy
   ```

O Caddy obtém e renova o certificado HTTPS automaticamente. Seus dados e os dados do Home Assistant e Mosquitto são mantidos em volumes Docker. A interface HTTP do Home Assistant fica acessível apenas na própria VPS pela porta 8123; o broker não publica sua porta na VPS.

## Configurar o Home Assistant

Para o acesso inicial, abra um túnel a partir do seu computador:

```bash
ssh -L 8123:127.0.0.1:8123 usuario@IP_DA_VPS
```

Mantenha o túnel aberto, acesse `http://localhost:8123` e conclua a configuração inicial. Em **Configurações → Sistema → Rede**, na seção do servidor HTTP, ative **Confiar em X-Forwarded-For** e adicione `172.28.50.0/24` aos **Proxies confiáveis**. Salve e confirme as configurações após o reinício. Essa faixa concede confiança aos contêineres conectados à rede do projeto. Se ela mudar no Compose, atualize-a também no Home Assistant.

Depois, acesse o domínio configurado usando HTTPS. Use uma senha forte e ative a autenticação em duas etapas no Home Assistant.

Em **Configurações → Dispositivos e serviços**, adicione a integração MQTT com estes dados:

| Campo | Valor |
| --- | --- |
| Broker | `mosquitto` |
| Porta | `1883` |
| Usuário | `homeassistant` |
| Senha | A senha criada acima |

O nome `mosquitto` é resolvido pela rede do Compose. Para acompanhar a atividade, use a opção de escutar o tópico `casa/energia` na integração MQTT.
