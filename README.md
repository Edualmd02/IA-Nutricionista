import streamlit as st
import json
import os
import random
from datetime import datetime, timedelta

st.set_page_config(page_title="IA Nutricionista", layout="wide")

# ======================
# ARQUIVO DE DADOS
# ======================
DATA_FILE = "user_data.json"

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"profile": {}, "selected_foods": [], "logs": {}, "weekly_menu": {}}

def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

data = load_data()

# ======================
# BANCO DE ALIMENTOS (completo)
# ======================
FOOD_DB = {
    "Café da manhã": {
        "Carboidratos": {"Pão francês": 270,"Pão integral": 250,"Aveia": 389,"Tapioca": 330,"Cuscuz": 112,"Panqueca": 220,"Granola": 471,"Cereal integral": 370},
        "Proteínas": {"Ovo": 155,"Clara de ovo": 52,"Queijo branco": 264,"Queijo minas": 243,"Iogurte natural": 59,"Iogurte grego": 120,"Leite desnatado": 34,"Leite integral": 60,"Whey protein": 400},
        "Frutas": {"Banana": 89,"Maçã": 52,"Mamão": 43,"Morango": 32,"Pera": 57,"Uva": 69,"Abacate": 160,"Manga": 60,"Melão": 34,"Abacaxi": 50},
        "Gorduras": {"Manteiga": 717,"Requeijão": 257,"Pasta de amendoim": 588,"Azeite de oliva": 884,"Creme de ricota": 174}
    },
    "Almoço": {
        "Carboidratos": {"Arroz branco": 130,"Arroz integral": 124,"Batata inglesa": 77,"Batata doce": 86,"Inhame": 97,"Macarrão": 131,"Cuscuz marroquino": 112,"Quinoa": 120,"Farofa": 365},
        "Proteínas bovinas": {"Alcatra": 250,"Patinho": 219,"Acém": 215,"Maminha": 242,"Coxão mole": 219,"Coxão duro": 223,"Fraldinha": 250,"Picanha": 290,"Cupim": 330,"Costela bovina": 340},
        "Proteínas suínas": {"Lombo suíno": 242,"Pernil suíno": 260,"Costelinha suína": 321,"Barriga de porco": 518,"Bisteca suína": 250},
        "Proteínas aves/peixes": {"Peito de frango": 165,"Coxa de frango": 215,"Sobrecoxa": 229,"Peito de peru": 135,"Tilápia": 96,"Salmão": 208,"Atum": 132,"Sardinha": 208},
        "Leguminosas": {"Feijão": 127,"Lentilha": 116,"Grão-de-bico": 164,"Ervilha": 81},
        "Verduras e legumes": {"Alface": 15,"Tomate": 18,"Tomate cereja": 18,"Cenoura": 41,"Abóbora": 26,"Abobrinha": 17,"Brócolis": 34,"Couve": 49,"Espinafre": 23,"Cebola roxa": 40,"Pepino": 16,"Beterraba": 43,"Vagem": 31,"Chuchu": 19,"Repolho": 25},
        "Purês": {"Purê de batata": 90,"Purê de batata doce": 95,"Purê de inhame": 102,"Purê de abóbora": 40,"Purê de mandioquinha": 120},
        "Gorduras": {"Azeite de oliva": 884,"Manteiga": 717,"Óleo de coco": 862}
    },
    "Lanche": {
        "Proteínas": {"Iogurte": 59,"Iogurte grego": 120,"Whey protein": 400,"Queijo cottage": 98,"Ricota": 174},
        "Oleaginosas": {"Castanha de caju": 553,"Castanha do pará": 659,"Amendoim": 567,"Nozes": 654,"Amêndoas": 579},
        "Frutas": {"Banana": 89,"Maçã": 52,"Pera": 57,"Uva": 69,"Morango": 32,"Abacate": 160,"Manga": 60,"Kiwi": 61,"Laranja": 47,"Tangerina": 53},
        "Carboidratos": {"Pão integral": 250,"Torrada integral": 380,"Biscoito integral": 430,"Granola": 471}
    },
    "Jantar": {
        "Proteínas": {"Peito de frango": 165,"Ovo": 155,"Tilápia": 96,"Salmão": 208,"Carne moída": 250,"Hambúrguer caseiro": 270},
        "Verduras e legumes": {"Alface": 15,"Tomate": 18,"Cenoura": 41,"Abobrinha": 17,"Brócolis": 34,"Couve-flor": 25,"Espinafre": 23,"Cogumelos": 22,"Berinjela": 25},
        "Carboidratos": {"Batata doce": 86,"Arroz integral": 124,"Quinoa": 120,"Purê de legumes": 60,"Macarrão integral": 124},
        "Gorduras": {"Azeite de oliva": 884,"Manteiga": 717}
    }
}

def flatten_food_db():
    flat = {}
    for meal, cats in FOOD_DB.items():
        for cat, items in cats.items():
            for food, kcal in items.items():
                if food in flat:
                    flat[food]["meal"].append(meal)
                else:
                    flat[food] = {"kcal": kcal, "meal":[meal], "category": cat}
    return flat

FLAT_FOODS = flatten_food_db()

# ======================
# CÁLCULOS NUTRICIONAIS
# ======================
def calculate_bmr(weight, height, age, gender):
    return 10*weight + 6.25*height - 5*age + (5 if gender=="Masculino" else -161)

def activity_multiplier(level):
    return {"Sedentário":1.2,"Levemente ativo":1.375,"Moderadamente ativo":1.55,"Muito ativo":1.725,"Extremamente ativo":1.9}[level]

def goal_adjustment(goal):
    return {"Emagrecer":-500,"Manter peso":0,"Ganhar massa":300}[goal]

def calculate_macros(calories):
    protein = calories*0.3/4
    carbs = calories*0.45/4
    fat = calories*0.25/9
    return protein, carbs, fat

def generate_daily_meals(calories, selected_foods):
    meals = {"Café da manhã":[],"Almoço":[],"Lanche":[],"Jantar":[]}
    split = {"Café da manhã":0.25,"Almoço":0.35,"Lanche":0.15,"Jantar":0.25}
    for meal in meals:
        allowed = [f for f in selected_foods if meal in FLAT_FOODS[f]["meal"]]
        if not allowed: continue
        items = random.sample(allowed, min(len(allowed), random.randint(2,4)))
        meal_calories = calories*split[meal]
        per_item_cal = meal_calories/len(items)
        for food in items:
            grams = (per_item_cal/FLAT_FOODS[food]["kcal"])*100
            meals[meal].append((food, round(grams)))
    return meals

def generate_weekly_menu(calories, selected_foods):
    weekly = {}
    today = datetime.now().date()
    for i in range(7):
        day = today + timedelta(days=i)
        weekly[str(day)] = generate_daily_meals(calories, selected_foods)
    return weekly

# ======================
# LOGIN SIMULADO
# ======================
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False
if "username" not in st.session_state:
    st.session_state.username = ""

if not st.session_state.logged_in:
    st.markdown("<h1 style='text-align:center;color:green;'>🥗 IA Nutricionista</h1>", unsafe_allow_html=True)
    st.markdown("<p style='text-align:center;'>Simule login inserindo seu nome:</p>", unsafe_allow_html=True)
    username = st.text_input("Nome de usuário")
    if st.button("Entrar"):
        if username.strip():
            st.session_state.logged_in = True
            st.session_state.username = username
            st.experimental_rerun()
    st.stop()

# ======================
# MENU PRINCIPAL
# ======================
st.sidebar.title(f"👋 {st.session_state.username}")
menu = st.sidebar.radio("Menu", ["Perfil", "Selecionar Alimentos", "Cardápio Semanal", "Registrar Refeições", "Histórico"])

# ======================
# PERFIL
# ======================
if menu=="Perfil":
    st.header("👤 Perfil do Usuário")
    profile = data.get("profile",{})
    col1,col2 = st.columns([1,2])
    with col1:
        st.image("https://cdn-icons-png.flaticon.com/512/3135/3135715.png", width=120)
    with col2:
        name = st.text_input("Nome", profile.get("name",st.session_state.username))
        birth = st.date_input("Data de nascimento", datetime.strptime(profile.get("birth","2000-01-01"), "%Y-%m-%d"))
        weight = st.number_input("Peso (kg)", 30.0,300.0,float(profile.get("weight",70.0)))
        height = st.number_input("Altura (cm)",120.0,230.0,float(profile.get("height",170.0)))
        age = st.number_input("Idade",10,120,int(profile.get("age",25)))
        gender = st.selectbox("Sexo",["Masculino","Feminino"], index=0 if profile.get("gender","Masculino")=="Masculino" else 1)
        activity = st.selectbox("Nível de atividade", ["Sedentário","Levemente ativo","Moderadamente ativo","Muito ativo","Extremamente ativo"],
                                index=["Sedentário","Levemente ativo","Moderadamente ativo","Muito ativo","Extremamente ativo"].index(profile.get("activity","Sedentário")))
        goal = st.selectbox("Objetivo", ["Emagrecer","Manter peso","Ganhar massa"],
                            index=["Emagrecer","Manter peso","Ganhar massa"].index(profile.get("goal","Manter peso")))
    if st.button("Salvar perfil"):
        data["profile"] = {"name":name,"birth":str(birth),"weight":weight,"height":height,"age":age,
                           "gender":gender,"activity":activity,"goal":goal}
        save_data(data)
        st.success("Perfil salvo!")

# ======================
# SELECIONAR ALIMENTOS
# ======================
elif menu=="Selecionar Alimentos":
    st.header("🥗 Selecione os alimentos que você consome")
    selected_foods = st.multiselect("Alimentos disponíveis:", list(FLAT_FOODS.keys()), default=data.get("selected_foods",[]))
    if st.button("Salvar seleção"):
        data["selected_foods"] = selected_foods
        save_data(data)
        st.success("Alimentos salvos!")

# ======================
# CARDÁPIO SEMANAL
# ======================
elif menu=="Cardápio Semanal":
    st.header("📅 Cardápio da Semana")
    profile = data.get("profile",{})
    selected_foods = data.get("selected_foods",[])
    if not profile:
        st.warning("Preencha seu perfil primeiro.")
    elif not selected_foods:
        st.warning("Selecione os alimentos que você consome na aba 'Selecionar Alimentos'.")
    else:
        bmr = calculate_bmr(profile["weight"],profile["height"],profile["age"],profile["gender"])
        tdee = bmr*activity_multiplier(profile["activity"])+goal_adjustment(profile["goal"])
        protein, carbs, fat = calculate_macros(tdee)

        st.metric("🔥 Calorias diárias", f"{int(tdee)} kcal")
        st.metric("🥩 Proteína", f"{int(protein)} g")
        st.metric("🍞 Carboidratos", f"{int(carbs)} g")
        st.metric("🥑 Gorduras", f"{int(fat)} g")

        if "weekly_menu" not in data or st.button("🔄 Gerar novo cardápio"):
            data["weekly_menu"] = generate_weekly_menu(tdee, selected_foods)
            save_data(data)
        weekly_menu = data["weekly_menu"]
        for day, meals in weekly_menu.items():
            st.subheader(day)
            for meal, items in meals.items():
                st.write(f"**{meal}**")
                for food, grams in items:
                    st.write(f"• {food}: {grams} g")
