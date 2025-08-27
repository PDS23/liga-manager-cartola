
# 🚀 Guia de Integração da API do Cartola FC - Liga Manager FC

## 📋 Visão Geral

Este guia fornece instruções detalhadas para integrar dados reais do Cartola FC no Liga Manager FC usando a API não oficial, mas amplamente utilizada pela comunidade.

## 🔐 Autenticação: Sistema GLBID

### Como Funciona

A autenticação é baseada na obtenção de um `GLBID` (token de sessão da Globo) através do login na Conta Globo.

### Processo de Autenticação

1. **Endpoint de Login:**
   ```
   POST https://authx-api.globoid.globo.com/v1/auth/authenticate
   ```

2. **Payload da Requisição:**
   ```json
   {
     "captcha": "",
     "payload": {
       "email": "pedro.silva1996@yahoo.com",
       "password": "Globo.1501",
       "serviceId": 4728
     }
   }
   ```

3. **Extração do GLBID:**
   O `GLBID` é retornado no cabeçalho `Set-Cookie`:
   ```
   Set-Cookie: GLBID=132e9b1b824c71a3602d69c693ea3df5d564b7938705a4d6f456c393673794771634d4f7a54313670763555332d684d6f3759316c32a74575556f5a546d343130303753346e7a616a6171713866373869653638; Path=/; Domain=.globo.com;
   ```

## 🏗️ Arquitetura Recomendada

### Backend como Proxy Seguro

```
Frontend (React) → Backend (Node.js/Express) → API Cartola FC
                                             ↑
                                      Gerencia GLBID
```

### Vantagens:
- ✅ Credenciais seguras no backend
- ✅ GLBID gerenciado centralmente
- ✅ Rate limiting controlado
- ✅ Cache de dados
- ✅ Logs de auditoria

## 🛠️ Implementação Backend

### 1. Configuração do Servidor

```javascript
// server.js
const express = require('express')
const cors = require('cors')
const axios = require('axios')

const app = express()
app.use(cors())
app.use(express.json())

// Armazenamento temporário do GLBID (usar Redis em produção)
let currentGLBID = null
let glbidExpiry = null

// Credenciais (usar variáveis de ambiente)
const CARTOLA_CREDENTIALS = {
  email: process.env.CARTOLA_EMAIL || 'pedro.silva1996@yahoo.com',
  password: process.env.CARTOLA_PASSWORD || 'Globo.1501'
}
```

### 2. Serviço de Autenticação

```javascript
// services/auth.js
async function authenticateCartola() {
  try {
    const response = await axios.post(
      'https://authx-api.globoid.globo.com/v1/auth/authenticate',
      {
        captcha: "",
        payload: {
          email: CARTOLA_CREDENTIALS.email,
          password: CARTOLA_CREDENTIALS.password,
          serviceId: 4728
        }
      }
    )

    // Extrair GLBID do Set-Cookie
    const setCookieHeader = response.headers['set-cookie']
    if (setCookieHeader) {
      const glbidCookie = setCookieHeader.find(cookie => 
        cookie.startsWith('GLBID=')
      )
      
      if (glbidCookie) {
        const glbid = glbidCookie.split(';')[0].replace('GLBID=', '')
        currentGLBID = glbid
        glbidExpiry = Date.now() + (24 * 60 * 60 * 1000) // 24 horas
        
        console.log('✅ GLBID obtido com sucesso')
        return glbid
      }
    }
    
    throw new Error('GLBID não encontrado na resposta')
  } catch (error) {
    console.error('❌ Erro na autenticação:', error.message)
    throw error
  }
}

async function getValidGLBID() {
  if (!currentGLBID || Date.now() > glbidExpiry) {
    await authenticateCartola()
  }
  return currentGLBID
}

module.exports = { getValidGLBID }
```

### 3. Endpoints da API

```javascript
// routes/cartola.js
const { getValidGLBID } = require('../services/auth')

// Middleware para adicionar GLBID
async function addGLBID(req, res, next) {
  try {
    const glbid = await getValidGLBID()
    req.glbid = glbid
    next()
  } catch (error) {
    res.status(401).json({ error: 'Falha na autenticação' })
  }
}

// Dados do usuário
app.get('/api/cartola/user', addGLBID, async (req, res) => {
  try {
    const response = await axios.get('https://api.cartola.globo.com/auth/user', {
      headers: {
        'Cookie': `GLBID=${req.glbid}`,
        'X-GLB-Token': req.glbid
      }
    })
    res.json(response.data)
  } catch (error) {
    res.status(500).json({ error: 'Erro ao buscar dados do usuário' })
  }
})

// Time do usuário
app.get('/api/cartola/time', addGLBID, async (req, res) => {
  try {
    const response = await axios.get('https://api.cartola.globo.com/auth/time', {
      headers: {
        'Cookie': `GLBID=${req.glbid}`,
        'X-GLB-Token': req.glbid
      }
    })
    res.json(response.data)
  } catch (error) {
    res.status(500).json({ error: 'Erro ao buscar time' })
  }
})

// Ligas do usuário
app.get('/api/cartola/ligas', addGLBID, async (req, res) => {
  try {
    const response = await axios.get('https://api.cartola.globo.com/auth/ligas', {
      headers: {
        'Cookie': `GLBID=${req.glbid}`,
        'X-GLB-Token': req.glbid
      }
    })
    res.json(response.data)
  } catch (error) {
    res.status(500).json({ error: 'Erro ao buscar ligas' })
  }
})

// Endpoints públicos (sem autenticação)
app.get('/api/cartola/mercado/status', async (req, res) => {
  try {
    const response = await axios.get('https://api.cartola.globo.com/mercado/status')
    res.json(response.data)
  } catch (error) {
    res.status(500).json({ error: 'Erro ao buscar status do mercado' })
  }
})

app.get('/api/cartola/atletas/mercado', async (req, res) => {
  try {
    const response = await axios.get('https://api.cartola.globo.com/atletas/mercado')
    res.json(response.data)
  } catch (error) {
    res.status(500).json({ error: 'Erro ao buscar atletas' })
  }
})
```

## 🎯 Integração Frontend

### 1. Serviço de API

```typescript
// services/cartolaAPI.ts
const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001/api'

class CartolaAPI {
  private async request(endpoint: string, options?: RequestInit) {
    const response = await fetch(`${API_BASE_URL}/cartola${endpoint}`, {
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers
      },
      ...options
    })

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`)
    }

    return response.json()
  }

  // Dados do usuário
  async getUser() {
    return this.request('/user')
  }

  // Time do usuário
  async getTime() {
    return this.request('/time')
  }

  // Ligas do usuário
  async getLigas() {
    return this.request('/ligas')
  }

  // Status do mercado
  async getMercadoStatus() {
    return this.request('/mercado/status')
  }

  // Atletas disponíveis
  async getAtletas() {
    return this.request('/atletas/mercado')
  }

  // Pontuação dos atletas por rodada
  async getAtletasPontuados(rodada: number) {
    return this.request(`/atletas/pontuados/${rodada}`)
  }
}

export const cartolaAPI = new CartolaAPI()
```

### 2. Hook para Dados do Cartola

```typescript
// hooks/useCartola.ts
import { useState, useEffect } from 'react'
import { cartolaAPI } from '../services/cartolaAPI'
import toast from 'react-hot-toast'

export function useCartola() {
  const [user, setUser] = useState(null)
  const [time, setTime] = useState(null)
  const [ligas, setLigas] = useState([])
  const [mercadoStatus, setMercadoStatus] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    loadCartolaData()
  }, [])

  const loadCartolaData = async () => {
    try {
      setLoading(true)

      // Carregar dados em paralelo
      const [userData, timeData, ligasData, mercadoData] = await Promise.all([
        cartolaAPI.getUser().catch(() => null),
        cartolaAPI.getTime().catch(() => null),
        cartolaAPI.getLigas().catch(() => []),
        cartolaAPI.getMercadoStatus().catch(() => null)
      ])

      setUser(userData)
      setTime(timeData)
      setLigas(ligasData)
      setMercadoStatus(mercadoData)

      if (userData) {
        toast.success('Dados do Cartola carregados!', { icon: '⚽' })
      }
    } catch (error) {
      console.error('Erro ao carregar dados do Cartola:', error)
      toast.error('Erro ao conectar com o Cartola FC')
    } finally {
      setLoading(false)
    }
  }

  return {
    user,
    time,
    ligas,
    mercadoStatus,
    loading,
    refresh: loadCartolaData
  }
}
```

### 3. Componente de Integração

```typescript
// components/CartolaIntegration.tsx
import React from 'react'
import { useCartola } from '../hooks/useCartola'
import { Trophy, Users, TrendingUp, Zap } from 'lucide-react'

export const CartolaIntegration: React.FC = () => {
  const { user, time, ligas, mercadoStatus, loading, refresh } = useCartola()

  if (loading) {
    return (
      <div className="bg-white rounded-2xl p-6 shadow-xl">
        <div className="animate-pulse">
          <div className="h-4 bg-gray-200 rounded w-1/4 mb-4"></div>
          <div className="space-y-3">
            <div className="h-3 bg-gray-200 rounded"></div>
            <div className="h-3 bg-gray-200 rounded w-5/6"></div>
          </div>
        </div>
      </div>
    )
  }

  return (
    <div className="space-y-6">
      {/* Status da Conexão */}
      <div className="bg-gradient-to-r from-green-50 to-emerald-50 border border-green-200 rounded-2xl p-6">
        <div className="flex items-center space-x-3">
          <div className="w-3 h-3 bg-green-500 rounded-full animate-pulse"></div>
          <h3 className="font-bold text-green-800">
            Conectado ao Cartola FC
          </h3>
          <button
            onClick={refresh}
            className="ml-auto text-green-600 hover:text-green-800 transition-colors"
          >
            Atualizar
          </button>
        </div>
      </div>

      {/* Dados do Usuário */}
      {user && (
        <div className="bg-white rounded-2xl p-6 shadow-xl">
          <h3 className="text-xl font-bold mb-4 flex items-center">
            <Trophy className="w-5 h-5 mr-2 text-yellow-500" />
            Meu Time Cartola
          </h3>
          <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
            <div className="text-center">
              <p className="text-2xl font-bold text-blue-600">{user.nome}</p>
              <p className="text-sm text-gray-500">Cartoleiro</p>
            </div>
            <div className="text-center">
              <p className="text-2xl font-bold text-green-600">
                {time?.patrimonio?.toFixed(2) || '0.00'}
              </p>
              <p className="text-sm text-gray-500">Patrimônio</p>
            </div>
            <div className="text-center">
              <p className="text-2xl font-bold text-purple-600">
                {time?.pontos || 0}
              </p>
              <p className="text-sm text-gray-500">Pontos</p>
            </div>
            <div className="text-center">
              <p className="text-2xl font-bold text-orange-600">
                {ligas.length}
              </p>
              <p className="text-sm text-gray-500">Ligas</p>
            </div>
          </div>
        </div>
      )}

      {/* Status do Mercado */}
      {mercadoStatus && (
        <div className="bg-white rounded-2xl p-6 shadow-xl">
          <h3 className="text-xl font-bold mb-4 flex items-center">
            <TrendingUp className="w-5 h-5 mr-2 text-blue-500" />
            Status do Mercado
          </h3>
          <div className="flex items-center space-x-4">
            <div className={`px-4 py-2 rounded-full text-sm font-bold ${
              mercadoStatus.status_mercado === 1 
                ? 'bg-green-100 text-green-800' 
                : 'bg-red-100 text-red-800'
            }`}>
              {mercadoStatus.status_mercado === 1 ? 'Aberto' : 'Fechado'}
            </div>
            <div className="text-sm text-gray-600">
              Rodada {mercadoStatus.rodada_atual}
            </div>
          </div>
        </div>
      )}
    </div>
  )
}
```

## 🚀 Próximos Passos

### 1. Configurar Backend
```bash
# Instalar dependências
npm install express cors axios dotenv

# Criar arquivo .env
echo "CARTOLA_EMAIL=pedro.silva1996@yahoo.com" > .env
echo "CARTOLA_PASSWORD=Globo.1501" >> .env

# Executar servidor
node server.js
```

### 2. Configurar Frontend
```bash
# Adicionar variável de ambiente
echo "REACT_APP_API_URL=http://localhost:3001/api" > .env.local
```

### 3. Implementar Componentes
- ✅ Integrar hook `useCartola` nas páginas
- ✅ Adicionar componente `CartolaIntegration`
- ✅ Configurar refresh automático de dados
- ✅ Implementar cache local para melhor performance

## ⚠️ Considerações Importantes

### Segurança
- 🔒 Nunca expor credenciais no frontend
- 🔒 Usar HTTPS em produção
- 🔒 Implementar rate limiting
- 🔒 Validar todas as entradas

### Estabilidade
- ⚡ Cache de dados para reduzir requisições
- ⚡ Fallback para dados simulados
- ⚡ Monitoramento de saúde da API
- ⚡ Logs detalhados para debugging

### Escalabilidade
- 📈 Usar Redis para cache distribuído
- 📈 Implementar queue para requisições
- 📈 Monitorar performance
- 📈 Otimizar consultas frequentes

---

**Credenciais de Teste:**
- Email: pedro.silva1996@yahoo.com
- Senha: Globo.1501

**Status:** ✅ Pronto para implementação
**Última atualização:** Janeiro 2025
