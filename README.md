# discord.py-python-bot

Bot Discord desenvolvido em Python usando discord.py.

## Comandos

- !saldo
- !daily
- !trabalhar
- !loja
- !comprar
- !rank

## Instalação

1. Instale as dependências:

pip install -r requirements.txt

2. Configure o token do Discord.

3. Execute:

python bot.py


import discord
from discord.ext import commands, tasks
import json
import random
import datetime
from typing import Optional

================= CONFIGURAÇÃO =================

TOKEN =PREFIXO = "!"

intents = discord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(command_prefix=PREFIXO, intents=intents)

================= BANCO DE DADOS =================

class EconomiaDB:
def init(self):
self.arquivo = "economia.json"
self.carregar()

def carregar(self):  
    try:  
        with open(self.arquivo, "r", encoding="utf-8") as f:  
            self.data = json.load(f)  
    except FileNotFoundError:  
        self.data = {"usuarios": {}, "loja": {}}  
        self.salvar()  

def salvar(self):  
    with open(self.arquivo, "w", encoding="utf-8") as f:  
        json.dump(self.data, f, indent=4, ensure_ascii=False)  

def get_usuario(self, user_id: int):  
    user_id = str(user_id)  
    if user_id not in self.data["usuarios"]:  
        self.data["usuarios"][user_id] = {  
            "dinheiro": 100,  
            "inventario": [],  
            "ultimo_daily": None,  
            "daily_streak": 1,  
            "nitro_simulado": False  
        }  
        self.salvar()  
    return self.data["usuarios"][user_id]  

def atualizar_usuario(self, user_id: int, dados: dict):  
    self.data["usuarios"][str(user_id)] = dados  
    self.salvar()

db = EconomiaDB()

================= LOJA =================

if not db.data["loja"]:
db.data["loja"] = {
"nitro_basic": {
"nome": "🎈 Nitro Simulado",
"preco": 100,
"descricao": "Emblema de Nitro por 7 dias",
"duracao": 7
},
"nitro_boost": {
"nome": "🚀 Nitro Boost Simulado",
"preco": 200,
"descricao": "Emblema especial por 30 dias",
"duracao": 30
},
"vip_pass": {
"nome": "👑 Passe VIP",
"preco": 500,
"descricao": "Cargo VIP no servidor",
"duracao": 30
}
}
db.salvar()

================= EVENTOS =================

@bot.event
async def on_ready():
print(f"✅ {bot.user} online!")
verificar_nitro_expirar.start()

================= COMANDOS =================

@bot.command()
async def saldo(ctx, membro: Optional[discord.Member] = None):
membro = membro or ctx.author
user = db.get_usuario(membro.id)

embed = discord.Embed(  
    title=f"💰 Carteira de {membro.name}",  
    color=discord.Color.gold()  
)  
embed.add_field(name="Dinheiro", value=f"🪙 {user['dinheiro']}", inline=False)  

if user["nitro_simulado"]:  
    embed.add_field(name="Nitro", value="🎈 Ativo (Simulado)", inline=False)  

if user["inventario"]:  
    embed.add_field(  
        name="Inventário",  
        value=", ".join(user["inventario"]),  
        inline=False  
    )  

await ctx.send(embed=embed)

---------------- DAILY ----------------

@bot.command()
@commands.cooldown(1, 86400, commands.BucketType.user)
async def daily(ctx):
user = db.get_usuario(ctx.author.id)
hoje = datetime.date.today()

streak = 1  
if user["ultimo_daily"]:  
    ultimo = datetime.date.fromisoformat(user["ultimo_daily"])  
    if (hoje - ultimo).days == 1:  
        streak = min(7, user["daily_streak"] + 1)  

base = random.randint(100, 300)  
bonus = streak * 50  
total = base + bonus  

user["dinheiro"] += total  
user["ultimo_daily"] = hoje.isoformat()  
user["daily_streak"] = streak  

db.atualizar_usuario(ctx.author.id, user)  

embed = discord.Embed(  
    title="🎁 Daily Resgatado",  
    description=f"Você ganhou **{total} moedas**!",  
    color=discord.Color.green()  
)  
embed.add_field(name="Base", value=base)  
embed.add_field(name="Streak", value=f"x{streak} (+{bonus})")  
embed.add_field(name="Saldo", value=user["dinheiro"], inline=False)  

await ctx.send(embed=embed)

---------------- TRABALHAR ----------------

@bot.command()
@commands.cooldown(1, 3600, commands.BucketType.user)
async def trabalhar(ctx):
trabalhos = [
("Programador", 1500, 3000),
("Designer", 1000, 2500),
("Moderador", 800, 2000),
("Tester", 1200, 2800)
]

nome, minimo, maximo = random.choice(trabalhos)  
ganho = random.randint(minimo, maximo)  

user = db.get_usuario(ctx.author.id)  
user["dinheiro"] += ganho  
db.atualizar_usuario(ctx.author.id, user)  

await ctx.send(  
    f"💼 Você trabalhou como **{nome}** e ganhou **{ganho} moedas**!"  
)

---------------- LOJA ----------------

@bot.command()
async def loja(ctx):
embed = discord.Embed(
title="🛒 Loja",
color=discord.Color.purple()
)

for item_id, item in db.data["loja"].items():  
    embed.add_field(  
        name=f"{item['nome']} — 🪙 {item['preco']}",  
        value=f"{item['descricao']}\nID: `{item_id}`",  
        inline=False  
    )  

await ctx.send(embed=embed)

---------------- COMPRAR ----------------

@bot.command()
async def comprar(ctx, item_id: str = None):
if not item_id or item_id not in db.data["loja"]:
await ctx.send("❌ Item inválido.")
return

item = db.data["loja"][item_id]  
user = db.get_usuario(ctx.author.id)  

if user["dinheiro"] < item["preco"]:  
    await ctx.send("❌ Dinheiro insuficiente.")  
    return  

user["dinheiro"] -= item["preco"]  
mensagem = f"🎉 Você comprou **{item['nome']}**!"  

if item_id.startswith("nitro"):  
    user["nitro_simulado"] = True  
    user["nitro_expira"] = (  
        datetime.datetime.now() + datetime.timedelta(days=item["duracao"])  
    ).isoformat()  
    mensagem += "\n⚠️ Nitro simulado (não é real)."  

elif item_id == "vip_pass":  
    try:  
        vip = discord.utils.get(ctx.guild.roles, name="VIP")  
        if not vip:  
            vip = await ctx.guild.create_role(  
                name="VIP",  
                color=discord.Color.gold()  
            )  
        await ctx.author.add_roles(vip)  
        mensagem += "\n👑 Cargo VIP adicionado!"  
    except:  
        mensagem += "\n⚠️ Sem permissão para criar cargo."  

else:  
    user["inventario"].append(item["nome"])  

db.atualizar_usuario(ctx.author.id, user)  
await ctx.send(mensagem)

---------------- RANK ----------------

@bot.command()
async def rank(ctx):
ranking = []
for uid, data in db.data["usuarios"].items():
try:
user = await bot.fetch_user(int(uid))
ranking.append((user.name, data["dinheiro"]))
except:
pass

ranking.sort(key=lambda x: x[1], reverse=True)  

embed = discord.Embed(  
    title="🏆 Ranking",  
    color=discord.Color.gold()  
)  

for i, (nome, dinheiro) in enumerate(ranking[:10], 1):  
    embed.add_field(  
        name=f"{i}º {nome}",  
        value=f"🪙 {dinheiro}",  
        inline=False  
    )  

await ctx.send(embed=embed)

================= TASK =================

@tasks.loop(hours=24)
async def verificar_nitro_expirar():
agora = datetime.datetime.now()
mudou = False

for user in db.data["usuarios"].values():  
    if user.get("nitro_simulado") and "nitro_expira" in user:  
        if agora > datetime.datetime.fromisoformat(user["nitro_expira"]):  
            user["nitro_simulado"] = False  
            del user["nitro_expira"]  
            mudou = True  

if mudou:  
    db.salvar()

================= START =================

if TOKEN == "SEU_TOKEN_AQUI":
print("❌ Configure o TOKEN do bot!")
else:
bot.run(TOKEN)

Pode posta ele assim?

Eu quero posta ele na página de projetos no github.com