import logging
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, ContextTypes, filters
from transformers import pipeline
from openai import OpenAI

# Configuración de logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger("Sico_chatBot")

# Cliente de DeepSeek
deepseek_client = OpenAI(
    api_key="sk-dba9089c512942f0ac36776c3385d2e6",  # 🔐 Coloca aquí tu API Key
    base_url="https://api.deepseek.com"
)

# Cargar modelo de análisis de sentimiento
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")

# Palabras clave asociadas a riesgo emocional
HIGH_RISK_KEYWORDS = [
    "suicidio", "quiero morir", "matarme", "no puedo más", "todo está perdido",
    "depresión", "ansiedad", "odio vivir", "autolesion", "daño", "muerte", "sin salida"
]

MODERATE_RISK_KEYWORDS = [
    "triste", "vacío", "llorar", "sin esperanza", "estoy mal", "fracaso", "desmotivado", "cansado", "agotado", "mal"
]

# Firma del bot
BOT_SIGNATURE = "\n\n— *Sico_chatBot*, asistente emocional basado en inteligencia artificial.\n_Si te sientes en crisis, busca apoyo profesional inmediato._"

# Respuestas personalizadas según frases clave
CUSTOM_RESPONSES = {
    "necesito ayuda": (
        "💚 *Pedir ayuda es un acto de valentía.*\n\n"
        "Reconocer que necesitas apoyo ya es un paso enorme hacia tu bienestar. A veces, hablar con alguien puede ser el alivio que necesitas.\n"
        "Aquí estoy para escucharte y acompañarte en lo que estás sintiendo." + BOT_SIGNATURE
    ),
    "no tengo fuerzas": (
        "😞 *Lo entiendo, hay días en los que todo pesa más.*\n\n"
        "Recuerda que incluso el paso más pequeño es progreso. No estás solo/a, y mereces sentirte mejor. Estoy aquí contigo." + BOT_SIGNATURE
    ),
    "me siento solo": (
        "🤝 *Sentirse solo/a puede ser muy difícil.*\n\n"
        "Aunque ahora parezca que nadie entiende lo que vives, hay personas que se preocupan por ti. A veces, hablarlo ya aligera el corazón." + BOT_SIGNATURE
    ),
    "no valgo nada": (
        "💔 *Tu valor no depende de lo que sientas en este momento.*\n\n"
        "Eres importante, incluso si ahora no lo ves. Todos atravesamos oscuridad, pero siempre hay luz. Estoy contigo." + BOT_SIGNATURE
    )
}

# IA de DeepSeek
def get_ai_response(query: str) -> str:
    try:
        response = deepseek_client.chat.completions.create(
            model="deepseek-chat",
            messages=[
                {"role": "system", "content": "Eres un asistente emocional profesional, amable, empático y basado en principios psicológicos. Hablas en tono comprensivo y respetuoso."},
                {"role": "user", "content": query}
            ],
            stream=False
        )
        return response.choices[0].message.content
    except Exception as e:
        logger.error("Error con DeepSeek: %s", e)
        return "⚠️ Lo siento, ocurrió un error al procesar tu mensaje. Por favor, intenta de nuevo más tarde."

# Análisis de riesgo emocional
def analyze_risk(text: str) -> str | None:
    text = text.lower()

    # Riesgo alto
    if any(kw in text for kw in HIGH_RISK_KEYWORDS):
        return (
            "🚨 *Alerta de alto riesgo emocional detectada*\n\n"
            "Su mensaje contiene expresiones que podrían indicar un riesgo significativo para su bienestar emocional.\n"
            "Le recomendamos que busque atención inmediata por parte de un profesional de salud mental.\n\n"
            "📞 *Líneas de apoyo disponibles:*\n"
            "• Línea 123 (emergencias generales)\n"
            "• Línea 141 del ICBF (apoyo psicológico en Colombia)\n\n"
            "_Recuerde que no está solo/a y hay ayuda disponible._" +
            BOT_SIGNATURE
        )

    # Riesgo moderado
    if any(kw in text for kw in MODERATE_RISK_KEYWORDS):
        return (
            "⚠️ *Indicadores de malestar emocional identificados*\n\n"
            "Se ha detectado un posible estado emocional delicado. Es natural atravesar momentos difíciles, y reconocerlo es un primer paso importante.\n\n"
            "Le recomendamos considerar hablar con un profesional de salud mental o con alguien de confianza.\n"
            "_Estoy aquí para acompañarle y escucharle._ 💚" +
            BOT_SIGNATURE
        )

    # Análisis de sentimiento negativo
    sentiment = classifier(text)[0]
    if sentiment['label'] == 'NEGATIVE' and sentiment['score'] > 0.8:
        return (
            "⚠️ *Sentimiento negativo con alta probabilidad detectado*\n\n"
            "Su mensaje presenta una carga emocional negativa considerable. Es fundamental prestar atención a estas señales y no enfrentarlas en soledad.\n\n"
            "Considere buscar orientación profesional si estas emociones persisten.\n"
            "_Cuidar su salud mental es tan importante como cuidar su salud física._" +
            BOT_SIGNATURE
        )

    return None  # No hay alerta, se permite responder con IA

# Comando /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Bienvenido/a. Soy *Sico_chatBot*, un asistente emocional creado con criterios de inteligencia artificial y orientación psicológica básica.\n\n"
        "🧠 Estoy aquí para ayudarte a reflexionar sobre tu estado emocional y orientarte en caso de detectar posibles riesgos.\n"
        "💬 Puedes escribirme libremente. Analizaré tu mensaje con empatía y responsabilidad.\n\n"
        "⚠️ *Importante:* Este servicio no reemplaza atención psicológica profesional. En caso de crisis, contacta a un especialista o línea de emergencia.\n\n"
        "Cuando estés listo/a, puedes contarme cómo te sientes." +
        BOT_SIGNATURE,
        parse_mode="Markdown"
    )

# Manejador de mensajes
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_message = update.message.text.lower()
    logger.info("Mensaje recibido: %s", user_message)

    # Primero analizar si hay riesgo emocional
    risk_response = analyze_risk(user_message)
    if risk_response:
        await update.message.reply_text(risk_response, parse_mode="Markdown")
        return

    # Verificar si el mensaje coincide con una frase personalizada
    for key_phrase, custom_response in CUSTOM_RESPONSES.items():
        if key_phrase in user_message:
            await update.message.reply_text(custom_response, parse_mode="Markdown")
            return

    # Si no hay coincidencia ni alerta, responder con IA
    ai_response = get_ai_response(user_message)
    full_response = f"{ai_response.strip()}{BOT_SIGNATURE}"
    await update.message.reply_text(full_response, parse_mode="Markdown")

# Manejador de errores
async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE):
    logger.error("Error causado por %s: %s", update, context.error)

# Función principal
def main():
    token = '8156362888:AAHtVnRKQrhTS1sF42uwQ6Wd9QPs6odLICw'  # Reemplaza con tu token real
    app = Application.builder().token(token).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.add_error_handler(error_handler)

    print("🤖 Sico_chatBot está en línea...")
    app.run_polling()

if __name__ == "__main__":
    main()
