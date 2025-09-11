import discord
from discord import app_commands
import requests

# ====== CONFIG ======
PTERO_URL = "YOUR_PANEL_LINK"
PTERO_API_KEY = "YOUR_PTERO_API_KEY"
EGG_ID = 2
DISCORD_BOT_TOKEN = "YOUR_DISCORD_BOT_TOKEN"

# Admin IDs (Discord user IDs)
ADMIN_IDS = [1352979362139340822, 987654321098765432]

HEADERS = {
    "Authorization": f"Bearer {PTERO_API_KEY}",
    "Content-Type": "application/json",
    "Accept": "Application/vnd.pterodactyl.v1+json"
}

intents = discord.Intents.default()
intents.members = True

DISCORD_TO_PTERO = {}

def is_admin(interaction: discord.Interaction) -> bool:
    return interaction.user.id in ADMIN_IDS

class MyClient(discord.Client):
    def __init__(self):
        super().__init__(intents=intents)
        self.tree = app_commands.CommandTree(self)

    async def on_ready(self):
        print(f"✅ Logged in as {self.user}")
        try:
            await self.tree.sync()
            print("✅ Slash commands synced")
        except Exception as e:
            print(f"❌ Sync failed: {e}")

client = MyClient()

# ====== ADMIN COMMANDS ======

@client.tree.command(name="user_create", description="Create a Pterodactyl panel user and assign to a Discord user.")
@app_commands.describe(
    username="Panel username",
    email="Panel email",
    first_name="First name",
    last_name="Last name",
    password="Panel password",
    adminuser="Should this user be admin? True/False",
    assigned_to="Discord user to assign this account to"
)
async def user_create(
    interaction: discord.Interaction,
    username: str,
    email: str,
    first_name: str,
    last_name: str,
    password: str,
    adminuser: bool,
    assigned_to: discord.User
):
    if not is_admin(interaction):
        return await interaction.response.send_message("❌ Admins only.", ephemeral=True)

    await interaction.response.send_message("👤 Creating Pterodactyl user...", ephemeral=True)

    user_payload = {
        "username": username.lower().replace(" ", "_"),
        "email": email,
        "first_name": first_name,
        "last_name": last_name,
        "password": password,
        "root_admin": adminuser,
        "language": "en"
    }

    try:
        user_res = requests.post(f"{PTERO_URL}/api/application/users", headers=HEADERS, json=user_payload)
        if user_res.status_code != 201:
            return await interaction.followup.send(
                f"❌ User creation failed:\n```{user_res.text}```", ephemeral=True
            )

        user_id = user_res.json()["attributes"]["id"]
        DISCORD_TO_PTERO[assigned_to.id] = user_id

        await assigned_to.send(
            f"✅ Your panel account has been created!\n"
            f"🌐 Panel: {PTERO_URL}\n"
            f"📧 Email: `{email}`\n"
            f"🔑 Password: `{password}`\n"
            f"👤 Username: `{username}`\n"
            f"🆔 User ID: `{user_id}`\n"
            f"— Regards Arnav & Para"
        )

        await interaction.followup.send(
            f"✅ User created and assigned to {assigned_to.mention}. — Regards Arnav & Para",
            ephemeral=True
        )

    except Exception as e:
        return await interaction.followup.send(f"❌ Error creating user: {e}", ephemeral=True)


@client.tree.command(name="server_create", description="Create a Minecraft server and assign it to a user.")
@app_commands.describe(
    server_name="Server name",
    description="Server description",
    mc_version="Minecraft version",
    paper_build="Paper build number",
    ram="RAM in MB",
    cpu="CPU %",
    disk="Disk in MB",
    backups="Number of backups",
    allocations="Number of allocations",
    databases="Number of databases",
    creatinguser="Panel username",
    useremail="Panel email",
    password="Panel password",
    adminuser="Should this user be admin? True/False",
    assigned_to="Discord user"
)
async def server_create(
    interaction: discord.Interaction,
    server_name: str,
    description: str,
    mc_version: str,
    paper_build: str,
    ram: int,
    cpu: int,
    disk: int,
    backups: int,
    allocations: int,
    databases: int,
    creatinguser: str,
    useremail: str,
    password: str,
    adminuser: bool,
    assigned_to: discord.User
):
    if not is_admin(interaction):
        return await interaction.response.send_message("❌ Admins only.", ephemeral=True)

    await interaction.response.send_message("🔧 Creating user and server...", ephemeral=True)

    # Create user first
    user_payload = {
        "username": creatinguser.lower().replace(" ", "_"),
        "email": useremail,
        "first_name": creatinguser,
        "last_name": "pterobot",
        "password": password,
        "root_admin": adminuser,
        "language": "en"
    }

    try:
        user_res = requests.post(f"{PTERO_URL}/api/application/users", headers=HEADERS, json=user_payload)
        if user_res.status_code != 201:
            return await interaction.followup.send(f"❌ User creation failed:\n```{user_res.text}```", ephemeral=True)
        user_id = user_res.json()["attributes"]["id"]
        DISCORD_TO_PTERO[assigned_to.id] = user_id
    except Exception as e:
        return await interaction.followup.send(f"❌ Error creating user: {e}", ephemeral=True)

    # Build PaperMC JAR download URL
    jar_url = f"https://api.papermc.io/v2/projects/paper/versions/{mc_version}/builds/{paper_build}/downloads/paper-{mc_version}-{paper_build}.jar"

    # Server payload with Node ID 1
    server_payload = {
        "name": server_name,
        "description": description,
        "user": user_id,
        "egg": EGG_ID,
        "docker_image": "ghcr.io/pterodactyl/yolks:java_17",
        "startup": "java -Xms128M -Xmx{{SERVER_MEMORY}}M -jar {{SERVER_JARFILE}} nogui",
        "environment": {
            "SERVER_JARFILE": "server.jar",
            "DL_PATH": jar_url,
            "BUILD_NUMBER": "latest"
        },
        "limits": {"memory": ram, "swap": 0, "disk": disk, "io": 500, "cpu": cpu},
        "feature_limits": {"databases": databases, "allocations": allocations, "backups": backups},
        "node": 1,  # 👈 Always install on Node ID 1
        "start_on_completion": True
    }

    try:
        server_res = requests.post(f"{PTERO_URL}/api/application/servers", headers=HEADERS, json=server_payload)
        if server_res.status_code != 201:
            return await interaction.followup.send(f"❌ Server creation failed:\n```{server_res.text}```", ephemeral=True)
        server_id = server_res.json()["attributes"]["id"]

        await assigned_to.send(
            f"✅ Your server is ready!\n"
            f"🌐 Panel: {PTERO_URL}\n"
            f"📧 Email: `{useremail}`\n"
            f"🔑 Password: `{password}`\n"
            f"🖥 Server: `{server_name}`\n"
            f"🆔 Server ID: `{server_id}`\n"
            f"📍 Node: `1`\n"
            f"💾 Backups: `{backups}`, Allocations: `{allocations}`, Databases: `{databases}` — Regards Arnav & Para"
        )

        await interaction.followup.send(
            f"✅ Server created and DM sent to {assigned_to.mention}. — Regards Arnav & Para",
            ephemeral=True
        )
    except Exception as e:
        return await interaction.followup.send(f"❌ Server creation error: {e}", ephemeral=True)

# ====== RUN ======
client.run(DISCORD_BOT_TOKEN)
