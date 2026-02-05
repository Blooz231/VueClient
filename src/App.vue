<script setup>
import { ref, onMounted, nextTick } from 'vue';
import { 
  Send, 
  Bot, 
  User, 
  Plus, 
  MessageSquare, 
  History, 
  Settings,
  Zap,
  Loader2,
  Cpu,
  ChevronRight,
  LogOut
} from 'lucide-vue-next';
import axios from 'axios';

const messages = ref([]);
const inputValue = ref('');
const loading = ref(false);
const conversations = ref([]);
const currentConvUuid = ref(null);
const scrollRef = ref(null);

const API_URL = import.meta.env.VITE_API_BASE_URL;
const API_KEY = import.meta.env.VITE_API_KEY;

const scrollToBottom = async () => {
  await nextTick();
  if (scrollRef.value) {
    scrollRef.value.scrollIntoView({ behavior: 'smooth' });
  }
};

const fetchConversations = async () => {
  try {
    const resp = await axios.get(`${API_URL}/chat/conversations`, {
      headers: { 'X-API-Key': API_KEY }
    });
    conversations.value = resp.data.data || [];
  } catch (err) {
    console.error("Failed to fetch conversations", err);
  }
};

const startNewChat = () => {
  messages.value = [];
  currentConvUuid.value = null;
};

const selectConversation = async (uuid) => {
  currentConvUuid.value = uuid;
  loading.value = true;
  try {
    const resp = await axios.get(`${API_URL}/chat/conversations/${uuid}/messages`, {
      headers: { 'X-API-Key': API_KEY }
    });
    messages.value = (resp.data.messages || []).map(m => ({
      role: m.role,
      content: m.content
    }));
  } catch (err) {
    console.error("Failed to load messages", err);
  } finally {
    loading.value = false;
    scrollToBottom();
  }
};

const handleSend = async () => {
  if (!inputValue.value.trim()) return;

  const userMsg = { role: 'user', content: inputValue.value };
  messages.value.push(userMsg);
  const currentInput = inputValue.value;
  inputValue.value = '';
  loading.value = true;
  
  const assistantMsgIndex = messages.value.length;
  messages.value.push({ role: 'assistant', content: '' });

  let assistantContent = '';

  try {
    const response = await fetch(`${API_URL}/chat/stream`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': API_KEY,
        'Accept': 'text/event-stream'
      },
      body: JSON.stringify({
        message: currentInput,
        conversation_uuid: currentConvUuid.value
      })
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { value, done } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const dataStr = line.replace('data: ', '');
          if (dataStr === '[DONE]') continue;
          
          try {
            const data = JSON.parse(dataStr);
            if (data.text) {
              assistantContent += data.text;
              messages.value[assistantMsgIndex].content = assistantContent;
              scrollToBottom();
            }
          } catch (e) {
            // Partial
          }
        }
      }
    }
  } catch (err) {
    console.error("Chat error", err);
    messages.value[assistantMsgIndex].content = 'Erreur technique durant la transmission.';
  } finally {
    loading.value = false;
    fetchConversations();
  }
};

onMounted(() => {
  fetchConversations();
});
</script>

<template>
  <div class="flex h-screen w-full bg-background text-slate-200 overflow-hidden font-sans">
    <!-- Sidebar -->
    <aside class="w-80 bg-surface border-r border-white/5 flex flex-col hidden lg:flex shrink-0 h-full">
      <div class="p-6">
        <div class="flex items-center gap-3 mb-8">
          <div class="w-10 h-10 rounded-2xl bg-primary/10 flex items-center justify-center border border-primary/20 shadow-[0_0_20px_rgba(56,189,248,0.15)]">
            <Cpu :size="20" class="text-primary" />
          </div>
          <div>
            <h2 class="text-sm font-black uppercase tracking-[0.2em] text-white">Vue <span class="text-primary">Node</span></h2>
            <p class="text-[9px] font-black text-slate-700 uppercase tracking-widest">Distributed Hub</p>
          </div>
        </div>

        <button 
          @click="startNewChat"
          class="w-full flex items-center justify-between px-5 py-3.5 bg-white/[0.03] hover:bg-white/[0.08] border border-white/10 rounded-2xl transition-all group"
        >
          <div class="flex items-center gap-3">
            <Plus :size="18" class="text-primary group-hover:rotate-90 transition-transform duration-500" />
            <span class="text-xs font-black uppercase tracking-widest">Connect Buffer</span>
          </div>
        </button>
      </div>

      <nav class="flex-1 overflow-y-auto px-4 space-y-1 custom-scrollbar">
        <div class="px-4 text-[10px] font-black text-slate-800 uppercase tracking-[0.3em] mb-4 mt-4">Historique</div>
        <button
          v-for="conv in conversations" 
          :key="conv.uuid"
          @click="selectConversation(conv.uuid)"
          :class="['w-full flex items-center gap-4 px-4 py-3.5 rounded-2xl transition-all group', currentConvUuid === conv.uuid ? 'bg-primary/10 border border-primary/20' : 'hover:bg-white/[0.03] border border-transparent']"
        >
          <div :class="['w-1.5 h-1.5 rounded-full shrink-0', currentConvUuid === conv.uuid ? 'bg-primary animate-pulse shadow-[0_0_8px_rgba(56,189,248,0.8)]' : 'bg-slate-800']" />
          <div class="flex-1 truncate text-left">
            <p :class="['text-xs font-bold truncate transition-colors', currentConvUuid === conv.uuid ? 'text-white' : 'text-slate-500 group-hover:text-slate-300']">
              {{ conv.title || 'Inference Flow' }}
            </p>
            <span class="text-[8px] font-black text-slate-800 uppercase tracking-tighter block mt-1">{{ conv.updated_at_diff }}</span>
          </div>
          <ChevronRight :size="12" :class="['shrink-0 transition-opacity', currentConvUuid === conv.uuid ? 'opacity-100 text-primary' : 'opacity-0 group-hover:opacity-100 text-slate-800']" />
        </button>
      </nav>

      <div class="p-6">
        <div class="bg-white/[0.02] border border-white/5 rounded-3xl p-4 flex items-center gap-4 group hover:bg-white/[0.04] transition-all cursor-pointer">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-br from-primary to-sky-600 flex items-center justify-center font-black text-xs text-slate-900 shadow-lg">V</div>
          <div class="flex-1 min-w-0">
            <p class="text-[10px] font-black uppercase tracking-[0.1em] text-white">Vue Client</p>
            <p class="text-[8px] text-slate-700 font-bold uppercase tracking-tighter">System Link: Active</p>
          </div>
          <LogOut :size="14" class="text-slate-800 group-hover:text-slate-400" />
        </div>
      </div>
    </aside>

    <!-- Main Content -->
    <main class="flex-1 flex flex-col relative h-full bg-background overflow-hidden">
      <!-- Glow Gradients -->
      <div class="absolute top-0 right-0 w-[500px] h-[500px] bg-primary/5 blur-[120px] rounded-full pointer-events-none -mr-40 -mt-40"></div>
      
      <header class="h-20 flex items-center justify-between px-8 border-b border-white/5 backdrop-blur-xl bg-background/50 z-30 shrink-0">
        <div class="flex items-center gap-4">
          <div class="lg:hidden w-8 h-8 rounded-lg bg-primary/20 flex items-center justify-center border border-primary/30">
            <Cpu :size="16" class="text-primary" />
          </div>
          <div>
            <div class="flex items-center gap-2">
              <h1 class="text-xs font-black uppercase tracking-[0.3em] text-white">Akasi Core</h1>
              <div class="flex items-center gap-1.5 px-2 py-0.5 rounded-full bg-secondary/10 border border-secondary/20">
                <div class="w-1 h-1 rounded-full bg-secondary animate-pulse" />
                <span class="text-[7px] font-black text-secondary uppercase tracking-[0.2em]">Neural Up</span>
              </div>
            </div>
            <p class="text-[8px] text-slate-700 font-bold uppercase tracking-widest mt-1">Vue Implementation v1.0.4</p>
          </div>
        </div>
        <div class="flex items-center gap-3">
          <button class="p-2 hover:bg-white/5 rounded-lg transition-all text-slate-500 hover:text-white"><History :size="18" /></button>
          <button class="p-2 hover:bg-white/5 rounded-lg transition-all text-slate-500 hover:text-white group"><Settings :size="18" class="group-hover:rotate-90 transition-transform duration-500" /></button>
        </div>
      </header>

      <div class="flex-1 overflow-y-auto px-6 py-10 md:px-12 relative z-20 custom-scrollbar h-full">
        <div v-if="messages.length === 0" class="h-full flex flex-col items-center justify-center max-w-2xl mx-auto text-center py-20">
          <div class="relative mb-10 group">
            <div class="absolute inset-0 bg-primary/20 blur-[30px] rounded-full group-hover:bg-primary/30 transition-all duration-700"></div>
            <div class="relative w-24 h-24 rounded-[2rem] bg-gradient-to-br from-surface to-background border border-white/10 flex items-center justify-center shadow-2xl">
              <Bot :size="48" class="text-primary relative z-10" />
            </div>
          </div>
          <h2 class="text-xs font-black uppercase tracking-[0.5em] text-slate-800 mb-6 font-bold">Inference UI</h2>
          <h1 class="text-4xl font-black mb-8 tracking-tight text-white leading-tight">Accédez au <span class="text-primary italic">Brain Akasi.</span></h1>
          
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 w-full max-w-xl">
            <button 
              v-for="t in ['Analyse mes datas', 'Schéma système', 'Audit de sécurité', 'Résumé des logs']"
              :key="t"
              @click="inputValue = t"
              class="flex items-center gap-4 p-5 bg-white/[0.02] border border-white/5 rounded-2xl text-[10px] font-black uppercase tracking-widest text-slate-500 hover:text-white transition-all text-left group hover:bg-primary/5"
            >
              <span class="flex-1">{{ t }}</span>
              <ChevronRight :size="14" class="text-slate-800 opacity-0 group-hover:opacity-100 transition-opacity" />
            </button>
          </div>
        </div>

        <div v-else class="max-w-4xl mx-auto space-y-12 pb-24 h-full">
          <div 
            v-for="(msg, idx) in messages" 
            :key="idx"
            :class="['flex gap-6 group', msg.role === 'assistant' ? 'bg-white/[0.01] border border-white/[0.03] p-8 rounded-[2rem] relative overflow-hidden' : '']"
          >
            <div class="flex flex-col items-center shrink-0">
              <div :class="['w-11 h-11 rounded-2xl flex items-center justify-center border transition-all duration-500', 
                msg.role === 'assistant' ? 'bg-primary/20 border-primary/30 text-primary shadow-lg shadow-primary/10' : 'bg-surface border-white/10 text-slate-500']">
                <Bot v-if="msg.role === 'assistant'" :size="22" />
                <User v-else :size="22" />
              </div>
            </div>

            <div class="flex-1 min-w-0 py-1">
              <div class="flex items-center gap-2 mb-4">
                <span :class="['text-[10px] font-black uppercase tracking-[0.2em]', msg.role === 'assistant' ? 'text-primary' : 'text-slate-700']">
                  {{ msg.role === 'assistant' ? 'Neural Core' : 'User Node' }}
                </span>
              </div>
              <div class="text-[14px] font-medium leading-[1.8] text-slate-300 whitespace-pre-wrap">
                {{ msg.content }}
                <div v-if="loading && idx === messages.length - 1 && !msg.content" class="flex flex-col gap-4 mt-2">
                  <div class="flex items-center gap-3 text-primary">
                    <Loader2 :size="16" class="animate-spin" />
                    <span class="text-[10px] uppercase font-black tracking-[0.3em] animate-pulse">Computing...</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
          <div ref="scrollRef" class="h-4"></div>
        </div>
      </div>

      <footer class="p-8 md:p-12 bg-gradient-to-t from-background via-background to-transparent z-40 sticky bottom-0 shrink-0">
        <div class="max-w-4xl mx-auto relative group">
          <div class="absolute -inset-1 bg-gradient-to-r from-primary/20 to-sky-500/20 rounded-[2rem] blur opacity-0 group-focus-within:opacity-100 transition duration-1000"></div>
          <div class="relative flex items-end gap-3 bg-surface border border-white/10 p-4 pl-8 rounded-[2rem] shadow-2xl backdrop-blur-3xl">
            <textarea 
              v-model="inputValue"
              @keydown.enter.prevent="handleSend"
              placeholder="Signal d'entrée..."
              rows="1"
              class="flex-1 bg-transparent border-none outline-none py-3 text-sm font-medium resize-none max-h-48 text-white placeholder:text-slate-800"
            ></textarea>
            <button 
              @click="handleSend" 
              :disabled="loading || !inputValue.trim()"
              :class="['p-3.5 rounded-2xl transition-all duration-500', inputValue.trim() ? 'bg-primary text-slate-900 shadow-xl shadow-primary/20' : 'bg-white/5 text-slate-900']"
            >
              <Send :size="20" />
            </button>
          </div>
          <div class="flex justify-between items-center mt-6 px-4">
            <span class="text-[8px] font-black text-slate-800 uppercase tracking-widest">Protocol Sync: Active • 2026</span>
            <span class="text-[8px] font-black text-slate-800 uppercase tracking-widest leading-none">Akasi Intelligence Engine Engine</span>
          </div>
        </div>
      </footer>
    </main>
  </div>
</template>

<style>
/* Reset and utilities */
html, body, #app {
  height: 100%;
}

.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  @apply bg-white/5 rounded-full;
}
</style>
