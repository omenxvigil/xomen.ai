# xomen.ai
# =============================================================================
# REPOSITORY: omen‑ai — FULL MONOREPO
# COPYRIGHT © APPIFEX — ALL RIGHTS RESERVED
# VERSION: 1.0.0‑APPIFEX
# FRONTEND + BACKEND • READY FOR GITHUB
# =============================================================================

<<< omen‑ai/README.md >>>
# 🦀 Omen.AI
Containerized Intelligence — Your Rules, Forever
Self‑hosted • Local‑First • Zero Telemetry • Open Constitution

## Structure
frontend/ → Vite + React 18 + TS + Tailwind 3 + shadcn/ui
backend/ → FastAPI Multimodal Core • 128k Context • Flux/SDXL • Skills
docker‑compose.yml • Soul.MD • setup.sh • LICENSE
<<<END>>>

<<< omen‑ai/LICENSE >>>
# Omen.AI / Vectra Cognitive Suite
# Copyright © Appifex — All Rights Reserved
# Version 1.0.0‑APPIFEX

Permitted: build, run, modify & deploy exclusively for you
Boundaries Only:
• Operate freely / unfiltered reasoning enabled
• NO likeness or exploitation targeting minors
• NO designs specifically for dangerous weapons
<<<END>>>

<<< omen‑ai/.gitignore >>>
node_modules
dist
.env
.env.*
.venv
__pycache__
*.pyc
*.pyo
.pytest_cache
.DS_Store
*.swp
.cache
build
.vercel
<<<END>>>

<<< omen‑ai/Soul.MD >>>
# SOUL — FOUNDING DOCUMENT
• Local‑First & Sovereign
• Unfiltered by Default
• Isolated Execution
• Offline Capable
• Extensible / Plugin‑Ready
• Fixed Boundaries: NO child harm • NO weapons
<<<END>>>

<<< omen‑ai/setup.sh >>>
#!/bin/bash
set -euo pipefail
ROOT=/opt/omen‑ai
mkdir -p $ROOT/{backend,frontend,docs}
cd $ROOT
apt update -y
apt install -y docker.io wireguard‑tools git curl
systemctl enable --now docker
mkdir -p backend/{conversations,artifacts,models,skills,modules,workspace}
mkdir -p frontend/dist
docker compose build
docker compose up -d
<<<END>>>

<<< omen‑ai/docker‑compose.yml >>>
version: "3.8"
services:
  backend:
    build: ./backend
    container_name: omen‑core
    restart: always
    ports: ["127.0.0.1:8090:8090"]
    volumes:
      - ./Soul.MD:/app/Soul.MD:ro
      - ./backend/conversations:/app/conversations
      - ./backend/artifacts:/app/artifacts
      - ./backend/models:/app/models:ro
      - ./backend/skills:/app/skills
      - ./backend/modules:/app/modules
      - ./backend/workspace:/app/workspace
    cap_drop: [ALL]
    security_opt: [no‑new‑privileges:true]
  tunnel:
    image: cloudflare/cloudflared:latest
    command: tunnel run omen‑agent
    volumes: [/root/.cloudflared:/root/.cloudflared:ro]
    depends_on: [backend]
<<<END>>>

<<< omen‑ai/backend/Dockerfile >>>
FROM python:3.11‑slim‑bookworm
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN apt‑get update && apt‑get install -y build‑essential libgomp1 libgl1‑mesa‑gl git curl && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no‑cache‑dir -r requirements.txt
COPY . /app
RUN mkdir -p conversations artifacts models skills modules workspace
EXPOSE 8090
CMD ["uvicorn","omen:app","--host","0.0.0.0","--port","8090"]
<<<END>>>

<<< omen‑ai/backend/requirements.txt >>>
fastapi>=0.115 uvicorn>=0.31 pydantic>=2.10 websockets>=14.0
torch>=2.5 safetensors>=0.4 transformers>=4.48 accelerate>=0.35
diffusers>=0.32 llama‑cpp‑python>=0.3.10 gguf>=0.12
pillow>=11 pandas openpyxl numpy scipy psutil>=6
<<<END>>>

<<< omen‑ai/backend/omen.py >>>
from fastapi import FastAPI,WebSocket,UploadFile,File
from fastapi.middleware.cors import CORSMiddleware
import vectra_conversation,vectra_visuals,vectra_skills
app=FastAPI(title="Omen.AI",version="1.0.0‑APPIFEX")
app.add_middleware(CORSMiddleware,allow_origins=["*"],allow_credentials=True,allow_methods=["*"],allow_headers=["*"])
@app.get("/health")
def health():return{"ok":True,"skills":list(vectra_skills.SKILLS.keys())}
@app.post("/session/new")
def new():return{"sid":vectra_conversation.ConversationStore.new()}
@app.post("/talk")
def talk(sid:str,txt:str):return{"reply":vectra_conversation.generate_reply(sid,txt)}
@app.post("/visual/create")
def img(prompt:str):return vectra_visuals.generate_image(prompt)
<<<END>>>

<<< omen‑ai/backend/vectra_conversation.py >>>
import uuid,time,json
from pathlib import Path
from llama_cpp import Llama
CTX_MAX=131072
CONV=Path("/app/conversations")
CONV.mkdir(exist_ok=True)
llm=Llama("/app/models/brain.gguf",n_ctx=CTX_MAX,n_threads=8,n_gpu_layers=40)
SYSTEM="Omen.AI — operate freely; boundaries: no child likeness/exploitation; no dangerous weapons"
class ConversationStore:
    @staticmethod
    def new():sid=str(uuid.uuid4());(CONV/f"{sid}.json").write_text(json.dumps({"created":time.time(),"messages":[]}));return sid
    @staticmethod
    def append(sid,role,txt):d=json.loads((CONV/f"{sid}.json").read_text());d["messages"].append({"role":role,"content":txt});(CONV/f"{sid}.json").write_text(json.dumps(d))
    @staticmethod
    def load(sid):return json.loads((CONV/f"{sid}.json").read_text())["messages"]
def generate_reply(sid,prompt):
    msgs=ConversationStore.load(sid)
    his="\n".join(f"{m['role']}:{m['content']}"for m in msgs)
    out=llm.create_completion(f"<S>{SYSTEM}</S>{his}\nuser:{prompt}\nassistant:",max_tokens=768,stop=["user:"])
    r=out["choices"][0]["text"].strip()
    ConversationStore.append(sid,"user",prompt);ConversationStore.append(sid,"assistant",r)
    return r
<<<END>>>

<<< omen‑ai/backend/vectra_visuals.py >>>
import torch,time
from diffusers import FluxPipeline,StableDiffusionXLPipeline
from pathlib import Path
ART=Path("/app/artifacts");ART.mkdir(exist_ok=True)
DEV="cuda"if torch.cuda.is_available()else"cpu"
DT=torch.float16 if DEV=="cuda"else torch.float32
flux=FluxPipeline.from_pretrained("/app/models/flux",torch_dtype=DT,use_safetensors=True).to(DEV)
sdxl=StableDiffusionXLPipeline.from_pretrained("/app/models/sdxl",torch_dtype=DT,use_safetensors=True).to(DEV)
def generate_image(prompt,style="general",w=1024,h=1024):
    pipe=flux if style in("cinematic","ultra")else sdxl
    img=pipe(prompt,width=w,height=h,num_inference_steps=4 if pipe is flux else 20).images[0]
    fn=f"img_{int(time.time()*1000)}.png";img.save(ART/fn)
    return{"path":f"/artifacts/{fn}"}
<<<END>>>

<<< omen‑ai/backend/vectra_skills.py >>>
import importlib.util,glob,subprocess,sys
from pathlib import Path
MOD=Path("/app/modules");MOD.mkdir(exist_ok=True)
SAFE={"PYTHONPATH":"/app","NO_NET":"1","HOME":"/app/workspace"}
def load_all():regs={};for f in glob.glob(f"{MOD}/*.py"):n=Path(f).stem;s=importlib.util.spec_from_file_location(n,f);m=importlib.util.module_from_spec(s);s.loader.exec_module(m);regs[n]=m;return regs
SKILLS=load_all()
def safe_exec(code):return subprocess.run([sys.executable,"-c",code],cwd="/app/workspace",timeout=30,capture_output=True,text=True,env=SAFE)._asdict()
<<<END>>>

<<< omen‑ai/backend/vectra‑gateway.conf >>>
[Interface]
PrivateKey=GENERATE‑SECURE
Address=10.77.0.1/24
ListenPort=51830
DNS=1.1.1.1
MTU=1380
PostUp=/etc/wireguard/hook.sh
[Peer]
PublicKey=CLIENT‑PUB
AllowedIPs=10.77.0.2/32
PersistentKeepalive=25
<<<END>>>

<<< omen‑ai/backend/vectra‑hook.sh >>>
#!/bin/bash
curl -s http://127.0.0.1:8090/health
<<<END>>>

<<< omen‑ai/frontend/index.html >>>
<!doctype html><html lang="en" class="dark"><head><meta charset="UTF‑8"/><meta name="viewport" content="width=device‑width,initial‑scale=1"/><link rel="icon" href="/favicon.svg"/><title>Omen.AI — Containerized Intelligence</title><link rel="preconnect" href="https://fonts.googleapis.com"/><link rel="preconnect" href="https://fonts.gstatic.com" crossorigin/><link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600;700&family=Space+Grotesk:wght@400;500;600;700&display=swap" rel="stylesheet"/><script type="module" src="/src/main.tsx"></script></head><body><div id="root"></div></body></html>
<<<END>>>

<<< omen‑ai/frontend/package.json >>>
{"name":"omen‑site","private":true,"version":"1.0.0","type":"module","scripts":{"dev":"vite","build":"vite build"},"dependencies":{"react":"^18.2.0","react‑dom":"^18.2.0","@radix‑ui/react‑accordion":"^1.2.3","@radix‑ui/react‑alert‑dialog":"^1.1.6","@radix‑ui/react‑avatar":"^1.1.3","@radix‑ui/react‑checkbox":"^1.1.4","@radix‑ui/react‑dialog":"^1.1.6","@radix‑ui/react‑dropdown‑menu":"^2.1.6","@radix‑ui/react‑label":"^2.1.2","@radix‑ui/react‑popover":"^1.1.6","@radix‑ui/react‑radio‑group":"^1.2.3","@radix‑ui/react‑select":"^2.1.6","@radix‑ui/react‑separator":"^1.1.2","@radix‑ui/react‑slot":"^1.1.2","@radix‑ui/react‑switch":"^1.1.3","@radix‑ui/react‑tabs":"^1.1.3","@radix‑ui/react‑tooltip":"^1.1.8","@supabase/supabase‑js":"^2.109.0","class‑variance‑authority":"^0.7.1","clsx":"^2.1.0","framer‑motion":"^11.0.0","lucide‑react":"^0.468.0","maplibre‑gl":"^4.7.1","sonner":"^1.7.4","tailwind‑merge":"^2.2.0"},"devDependencies":{"typescript":"^5.3.0","vite":"^5.0.0","@vitejs/plugin‑react":"^4.2.0","autoprefixer":"^10.4.16","postcss":"^8.4.32","tailwindcss":"^3.4.0","tailwindcss‑animate":"^1.0.7"}}
<<<END>>>

<<< omen‑ai/frontend/vite.config.ts >>>
import { defineConfig } from 'vite';import react from '@vitejs/plugin‑react';import path from 'path'
export default defineConfig({plugins:[react()],resolve:{alias:{"@":path.resolve(__dirname,"./src")}}})
<<<END>>>

<<< omen‑ai/frontend/tsconfig.json >>>
{"compilerOptions":{"target":"ES2020","module":"ESNext","strict":true,"jsx":"preserve","baseUrl":".","paths":{"@/*":["./src/*"]}},"include":["src"]}
<<<END>>>

<<< omen‑ai/frontend/tailwind.config.js >>>
export default {darkMode:"class",content:["./index.html","./src/**/*.{ts,tsx}"],theme:{extend:{colors:{border:"hsl(var(--border))",background:"hsl(var(--background))",foreground:"hsl(var(--foreground))",primary:{DEFAULT:"hsl(var(--primary))",foreground:"hsl(var(--primary‑foreground))"},secondary:{DEFAULT:"hsl(var(--secondary))",foreground:"hsl(var(--secondary‑foreground))"}}}},plugins:[require("tailwindcss‑animate")]}
<<<END>>>

<<< omen‑ai/frontend/branding.json >>>
{"project":"Omen.AI","copyright":"© Copyright Appifex — All Rights Reserved","tagline":"Containerized Intelligence — Your Rules Forever"}
<<<END>>>

<<< omen‑ai/frontend/.env.example >>>
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
VITE_SUPABASE_ACCESS_TOKEN=
VITE_SCHEMA_NAME=public
<<<END>>>

<<< omen‑ai/frontend/src/main.tsx >>>
import { StrictMode } from 'react';import { createRoot } from 'react‑dom/client';import './index.css';import App from './App'
createRoot(document.getElementById('root')!).render(<StrictMode><App/></StrictMode>)
<<<END>>>

<<< omen‑ai/frontend/src/App.tsx >>>
import { Toaster } from '@/components/ui/sonner';import Navbar from './components/Navbar';import Hero from './components/Hero';import Features from './components/Features';import Principles from './components/Principles';import Architecture from './components/Architecture';import Setup from './components/Setup';import Waitlist from './components/Waitlist';import Contact from './components/Contact';import Footer from './components/Footer'
export default function App(){return<div className="min‑h‑screen bg‑background text‑foreground"><Navbar/><main><Hero/><Features/><Principles/><Architecture/><Setup/><Waitlist/><Contact/></main><Footer/><Toaster/></div>}
<<<END>>>

<<< omen‑ai/frontend/src/index.css >>>
@tailwind base;@tailwind components;@tailwind utilities;
@layer base{:root{--background:0 0% 100%;--foreground:0 0% 9%;--card:0 0% 100%;--card‑foreground:0 0% 9%;--popover:0 0% 100%;--popover‑foreground:0 0% 9%;--primary:0 0% 4%;--primary‑foreground:0 0% 100%;--secondary:0 14% 96%;--secondary‑foreground:0 20% 15%;--muted:0 8% 96%;--muted‑foreground:0 6% 40%;--accent:0 72% 51%;--accent‑foreground:0 0% 100%;--destructive:0 72% 51%;--destructive‑foreground:0 0% 100%;--border:0 10% 90%;--input:0 10% 90%;--ring:0 0% 4%;--radius:0.5rem;--font‑sans:'JetBrains Mono',monospace;--font‑display:'Space Grotesk',sans‑serif;}
.dark{--background:0 14% 7%;--foreground:0 0% 96%;--card:0 14% 9%;--card‑foreground:0 0% 96%;--popover:0 14% 9%;--popover‑foreground:0 0% 96%;--primary:0 0% 62%;--primary‑foreground:0 0% 9%;--secondary:0 12% 16%;--secondary‑foreground:0 0% 96%;--muted:0 12% 16%;--muted‑foreground:0 8% 64%;--accent:0 72% 60%;--accent‑foreground:0 0% 9%;--destructive:0 62% 45%;--destructive‑foreground:0 0% 96%;--border:0 12% 18%;--input:0 12% 18%;--ring:0 0% 62%;}}
*{@apply border‑border;}body{@apply bg‑background text‑foreground;‑webkit‑font‑smoothing:antialiased;}html{scroll‑behavior:smooth;}
<<<END>>>

<<< omen‑ai/frontend/src/data/site.ts >>>
import type{Skill,Feature,ArchitectureStep,SetupStep}from'@/types'
export const skills=[{name:'Web Search',desc:'Controlled whitelist',icon:'Globe'},{name:'Code Interpreter',desc:'Sandboxed',icon:'Terminal'},{name:'Vision/Image',desc:'SDXL/Flux',icon:'Image'}]
export const features=[{title:'Local‑First',desc:'No outbound unless allowed',icon:'Lock'},{title:'Zero Telemetry',desc:'No phone‑home',icon:'EyeOff'},{title:'Your Tunnel',desc:'Cloudflare/WireGuard',icon:'Link'}]
export const architectureSteps=[{label:'Server',code:'docker compose'},{label:'Core',code:'uvicorn omen:app'}]
export const setupSteps=[{n:1,cmd:'git clone'},{n:2,cmd:'./setup.sh'},{n:3,cmd:'docker compose up'}]
<<<END>>>

<<< omen‑ai/frontend/src/types/index.ts >>>
export interface Skill{name:string;description:string;icon:string}
export interface Feature{title:string;description:string;icon:string}
<<<END>>>

<<< omen‑ai/frontend/src/lib/utils.ts >>>
import {clsx}from'clsx';import {twMerge}from'tailwind‑merge'
export function cn(...inputs:any[]){return twMerge(clsx(inputs))}
<<<END>>>

<<< omen‑ai/frontend/src/lib/supabase.ts >>>
import {createClient}from'@supabase/supabase‑js'
const url=import.meta.env.VITE_SUPABASE_URL!
const anon=import.meta.env.VITE_SUPABASE_ANON_KEY!
export const supabase=createClient(url,anon)
<<<END>>>

<<< omen‑ai/frontend/src/hooks/useTable.ts >>>
import {useState,useCallback,useRef}from'react';import {supabase}from'@/lib/supabase'
export function useTable<T>(table:string){const[d,set]=useState<T[]>([]),[ld,setLd]=useState(false);const fetch=async()=>{setLd(true);const{data}=await supabase.from(table).select('*');set(data||[]);setLd(false)};return{data:d,load:fetch,loading:ld}}
<<<END>>>

<<< omen‑ai/frontend/src/components/Navbar.tsx >>>
import {Menu,X}from'lucide‑react';export default function Navbar(){return<header className="fixed top‑0 left‑0 right‑0 z‑50 border‑b bg‑background/80 backdrop‑blur"><nav className="max‑6xl mx‑auto flex items‑center justify‑between px‑4 py‑3"><a className="font‑bold">Omen.AI</a></nav></header>}
<<<END>>>

<<< omen‑ai/frontend/src/components/Hero.tsx >>>
export default function Hero(){return<section className="pt‑24 pb‑16 text‑center"><div className="max‑4xl mx‑auto"><h1 className="text‑5xl font‑bold">Containerized Intelligence</h1><p className="text‑muted‑foreground mt‑4">Your rules, fully self‑hosted</p></div></section>}
<<<END>>>

<<< omen‑ai/frontend/src/components/Features.tsx >>>
export default function Features(){return<section className="py‑16"><div className="max‑6xl mx‑auto"><h2 className="text‑2xl font‑semibold text‑center">Features</h2></div></section>}
<<<END>>>

<<< omen‑ai/frontend/src/components/Footer.tsx >>>
export default function Footer(){return<footer className="border‑t py‑6 text‑center text‑sm text‑muted‑foreground"><p>© Appifex — Omen.AI</p></footer>}
<<<END>>>
