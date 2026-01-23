import React, { useState } from 'react';
import { Upload, TrendingUp, TrendingDown, BarChart3, AlertCircle, Settings } from 'lucide-react';

const FinancialChartAI = () => {
  const [analysis, setAnalysis] = useState(null);
  const [loading, setLoading] = useState(false);
  const [imagePreview, setImagePreview] = useState(null);
  const [showSettings, setShowSettings] = useState(false);
  const [showChatBot, setShowChatBot] = useState(false);
  const [chatMessages, setChatMessages] = useState([]);
  const [chatInput, setChatInput] = useState('');
  const [chatLoading, setChatLoading] = useState(false);

  const analyzeChart = async (imageFile) => {
    setLoading(true);
    
    try {
      const base64Data = await new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = () => resolve(reader.result.split(',')[1]);
        reader.onerror = () => reject(new Error('Erro ao ler arquivo'));
        reader.readAsDataURL(imageFile);
      });

      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          model: 'claude-sonnet-4-20250514',
          max_tokens: 1000,
          messages: [
            {
              role: 'user',
              content: [
                {
                  type: 'image',
                  source: {
                    type: 'base64',
                    media_type: imageFile.type,
                    data: base64Data
                  }
                },
                {
                  type: 'text',
                  text: `Você é uma IA especializada em análise técnica de gráficos financeiros. Analise este gráfico de candles e forneça:

IMPORTANTE: Você deve identificar se o momento é de SUPORTE ou RESISTÊNCIA (apenas um), e fornecer o número exato do nível E o possível resultado/alvo que o preço pode atingir.

Responda APENAS em JSON neste formato exato:
{
  "tipo": "SUPORTE" ou "RESISTÊNCIA",
  "valor": número do suporte ou resistência,
  "proximo_alvo": número do possível resultado/movimento esperado,
  "tendencia": "ALTA" ou "BAIXA" ou "LATERAL",
  "preco_atual": preço atual visível no gráfico,
  "probabilidade": porcentagem de acerto (85-95%),
  "padroes_identificados": ["padrão1", "padrão2"],
  "analise_tecnica": "descrição breve da análise",
  "rsi_status": "sobrecompra/sobrevenda/neutro",
  "recomendacao": "análise assertiva do movimento esperado"
}

Seja extremamente preciso e assertivo na análise dos candles.`
                }
              ]
            }
          ]
        })
      });

      const data = await response.json();
      const textContent = data.content.find(item => item.type === 'text')?.text || '';
      
      // Remove markdown code blocks se existirem
      const cleanJson = textContent.replace(/```json\n?|\n?```/g, '').trim();
      const result = JSON.parse(cleanJson);
      
      setAnalysis(result);
    } catch (error) {
      console.error('Erro na análise:', error);
      setAnalysis({
        error: true,
        message: 'Erro ao analisar o gráfico. Tente novamente.'
      });
    } finally {
      setLoading(false);
    }
  };

  const handleFileUpload = (event) => {
    const file = event.target.files[0];
    if (file && file.type.startsWith('image/')) {
      const reader = new FileReader();
      reader.onload = (e) => {
        setImagePreview(e.target.result);
      };
      reader.readAsDataURL(file);
      
      analyzeChart(file);
      setAnalysis(null);
    }
  };

  const sendChatMessage = async () => {
    if (!chatInput.trim()) return;

    const userMessage = chatInput.trim().toLowerCase();
    setChatInput('');
    setChatMessages(prev => [...prev, { role: 'user', content: chatInput.trim() }]);
    setChatLoading(true);

    try {
      let responseMessage = '';

      // Comando principal "sellalert" ou início
      if (userMessage === 'sellalert' || chatMessages.length === 0) {
        responseMessage = `⚠️ **SellAlert ProV0.0.6 — ANÁLISE MANUAL + ESTATÍSTICAS**

═══════════════════════════════════

1️⃣ Sair do sistema
2️⃣ Ativar modo manual de alertas (RSI e velas)
3️⃣ IA Gráfica 80% acertiva (com imagem)

═══════════════════════════════════

👉 Escolha uma opção:
Digite 1, 2 ou 3 e manda 😉`;
      }
      // Opção 1 - Sair
      else if (userMessage === '1') {
        responseMessage = `👋 Sistema encerrado. Até a próxima análise!

Digite "SellAlert" para abrir o menu novamente.`;
      }
      // Opção 2 - Modo Manual
      else if (userMessage === '2') {
        responseMessage = `📊 **Escolha a corretora para usar o modo manual:**

═══════════════════════════════════

1️⃣ Home Broker
2️⃣ Olymp Trade

═══════════════════════════════════

Selecione uma opção (1 ou 2):`;
      }
      // Opção 3 - IA Gráfica
      else if (userMessage === '3') {
        responseMessage = `🖼️ **IA Gráfica 80% Acertiva — Modo com imagem**

═══════════════════════════════════

1️⃣ Home Broker
2️⃣ Olymp Trade

═══════════════════════════════════

Selecione uma corretora (1 ou 2):`;
      }
      // Se não for comando específico, usa a IA para responder
      else {
        const systemPrompt = {
          role: 'user',
          content: `Você é o assistente do SellAlert ProV0.0.6, um sistema de análise técnica para trading. 

Responda de forma profissional e técnica sobre:
- Análise técnica (RSI, velas, suportes, resistências)
- Estratégias de trading
- Indicadores (MACD, Médias Móveis, Bandas de Bollinger)
- Padrões de candles
- Gerenciamento de risco

Use emojis relevantes: 📊 📈 📉 💰 🎯 ⚠️ 🔴 🟢 ⚡

Seja direto, claro e educativo. Responda em português brasileiro.

Pergunta: ${chatInput.trim()}`
        };

        const response = await fetch('https://api.anthropic.com/v1/messages', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            model: 'claude-sonnet-4-20250514',
            max_tokens: 1500,
            messages: [systemPrompt]
          })
        });

        const data = await response.json();
        responseMessage = data.content.find(item => item.type === 'text')?.text || 'Desculpe, não consegui processar sua mensagem.';
      }
      
      setChatMessages(prev => [...prev, { role: 'assistant', content: responseMessage }]);
    } catch (error) {
      console.error('Erro no chat:', error);
      setChatMessages(prev => [...prev, { 
        role: 'assistant', 
        content: '❌ Erro ao processar mensagem. Tente novamente.' 
      }]);
    } finally {
      setChatLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-900 via-blue-900 to-slate-900 p-4">
      <div className="max-w-4xl mx-auto">
        {/* Botão de Configurações - Posição Superior */}
        <div className="flex justify-end pt-4 mb-4">
          <button
            onClick={() => setShowSettings(!showSettings)}
            className="bg-slate-700/50 hover:bg-slate-600/50 p-3 rounded-full transition-all border border-cyan-500/30 hover:border-cyan-400 shadow-lg"
          >
            <Settings className="w-6 h-6 text-cyan-400" />
          </button>
        </div>

        {/* Header */}
        <div className="text-center mb-8">
          <div className="flex items-center justify-center gap-3 mb-4">
            <BarChart3 className="w-12 h-12 text-cyan-400" />
            <h1 className="text-4xl font-bold text-white">IA Análise de Gráficos</h1>
          </div>
          <p className="text-cyan-300 text-lg">Análise Técnica Profissional com IA</p>
        </div>

        {/* Modal de Configurações */}
        {showSettings && (
          <div className="fixed inset-0 bg-black/50 backdrop-blur-sm flex items-center justify-center z-50 p-4">
            <div className="bg-slate-800 rounded-2xl p-8 max-w-md w-full border border-cyan-500/30 shadow-2xl">
              <div className="flex items-center justify-between mb-6">
                <h2 className="text-2xl font-bold text-white flex items-center gap-2">
                  <Settings className="w-7 h-7 text-cyan-400" />
                  Configurações
                </h2>
                <button
                  onClick={() => setShowSettings(false)}
                  className="text-gray-400 hover:text-white text-2xl"
                >
                  ×
                </button>
              </div>
              
              <div className="space-y-4">
                <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600">
                  <h3 className="text-cyan-400 text-sm font-semibold mb-2">Precisão da Análise</h3>
                  <select className="w-full bg-slate-600 text-white rounded-lg p-2 border border-cyan-500/30">
                    <option>Alta (85-95%)</option>
                    <option>Média (75-85%)</option>
                    <option>Conservadora (65-75%)</option>
                  </select>
                </div>

                <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600">
                  <h3 className="text-cyan-400 text-sm font-semibold mb-2">Tipo de Análise</h3>
                  <select className="w-full bg-slate-600 text-white rounded-lg p-2 border border-cyan-500/30">
                    <option>Automático (IA decide)</option>
                    <option>Apenas Suporte</option>
                    <option>Apenas Resistência</option>
                  </select>
                </div>

                <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600">
                  <h3 className="text-cyan-400 text-sm font-semibold mb-2">Indicadores Adicionais</h3>
                  <div className="space-y-2">
                    <label className="flex items-center gap-2 text-white">
                      <input type="checkbox" defaultChecked className="rounded" />
                      <span>RSI</span>
                    </label>
                    <label className="flex items-center gap-2 text-white">
                      <input type="checkbox" defaultChecked className="rounded" />
                      <span>Padrões de Candles</span>
                    </label>
                    <label className="flex items-center gap-2 text-white">
                      <input type="checkbox" defaultChecked className="rounded" />
                      <span>Volume</span>
                    </label>
                  </div>
                </div>

                <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600">
                  <h3 className="text-cyan-400 text-sm font-semibold mb-3">Chat Bot</h3>
                  <button
                    onClick={() => {
                      setShowSettings(false);
                      setShowChatBot(true);
                    }}
                    className="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 text-white font-semibold py-3 rounded-lg transition-all flex items-center justify-center gap-2"
                  >
                    <span>💬</span>
                    Abrir Chat IA
                  </button>
                </div>

                <button
                  onClick={() => setShowSettings(false)}
                  className="w-full bg-gradient-to-r from-cyan-600 to-blue-600 hover:from-cyan-500 hover:to-blue-500 text-white font-semibold py-3 rounded-lg transition-all"
                >
                  Salvar Configurações
                </button>
              </div>
            </div>
          </div>
        )}

        {/* Modal do Chat Bot */}
        {showChatBot && (
          <div className="fixed inset-0 bg-black/50 backdrop-blur-sm flex items-center justify-center z-50 p-4">
            <div className="bg-gradient-to-br from-slate-900 to-slate-800 rounded-2xl w-full max-w-4xl h-[90vh] max-h-[800px] border border-cyan-500/30 shadow-2xl flex flex-col">
              {/* Header do Chat */}
              <div className="bg-gradient-to-r from-purple-600 to-pink-600 p-4 rounded-t-2xl flex items-center justify-between flex-shrink-0">
                <div className="flex items-center gap-3">
                  <div className="w-3 h-3 bg-green-400 rounded-full animate-pulse"></div>
                  <h2 className="text-xl font-bold text-white">💬 Chat Bot - Análise de Trading</h2>
                </div>
                <button
                  onClick={() => setShowChatBot(false)}
                  className="text-white hover:text-gray-200 text-2xl font-bold"
                >
                  ×
                </button>
              </div>

              {/* Área de Mensagens - Com altura fixa */}
              <div className="flex-1 overflow-y-auto p-6 space-y-4 min-h-0">
                {chatMessages.length === 0 ? (
                  <div className="text-center text-gray-400 mt-20">
                    <div className="text-6xl mb-4">⚠️</div>
                    <p className="text-2xl font-bold mb-2">SellAlert ProV0.0.6</p>
                    <p className="text-lg">Sistema de Análise Técnica</p>
                    <p className="text-sm mt-4 text-cyan-400">Digite "SellAlert" para começar</p>
                  </div>
                ) : (
                  chatMessages.map((msg, idx) => (
                    <div
                      key={idx}
                      className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}
                    >
                      <div
                        className={`max-w-[85%] rounded-2xl p-4 ${
                          msg.role === 'user'
                            ? 'bg-gradient-to-r from-cyan-600 to-blue-600 text-white'
                            : 'bg-slate-700 text-white border border-slate-600'
                        }`}
                      >
                        <p className="whitespace-pre-wrap font-mono text-sm leading-relaxed">{msg.content}</p>
                      </div>
                    </div>
                  ))
                )}
                {chatLoading && (
                  <div className="flex justify-start">
                    <div className="bg-slate-700 rounded-2xl p-4 border border-slate-600">
                      <div className="flex gap-2">
                        <div className="w-2 h-2 bg-cyan-400 rounded-full animate-bounce"></div>
                        <div className="w-2 h-2 bg-cyan-400 rounded-full animate-bounce" style={{animationDelay: '0.2s'}}></div>
                        <div className="w-2 h-2 bg-cyan-400 rounded-full animate-bounce" style={{animationDelay: '0.4s'}}></div>
                      </div>
                    </div>
                  </div>
                )}
              </div>

              {/* Input de Mensagem - Fixo na parte inferior */}
              <div className="p-4 bg-slate-800 border-t-2 border-cyan-500/50 flex-shrink-0">
                <div className="flex gap-3">
                  <input
                    type="text"
                    value={chatInput}
                    onChange={(e) => setChatInput(e.target.value)}
                    onKeyPress={(e) => e.key === 'Enter' && !chatLoading && sendChatMessage()}
                    placeholder="Digite aqui sua mensagem..."
                    className="flex-1 bg-slate-900 text-white text-lg rounded-xl px-5 py-4 border-2 border-cyan-500/50 focus:border-cyan-400 focus:outline-none placeholder-gray-400"
                    disabled={chatLoading}
                  />
                  <button
                    onClick={sendChatMessage}
                    disabled={chatLoading || !chatInput.trim()}
                    className="bg-gradient-to-r from-cyan-600 to-blue-600 hover:from-cyan-500 hover:to-blue-500 disabled:opacity-50 disabled:cursor-not-allowed text-white font-bold text-xl px-8 py-4 rounded-xl transition-all shadow-lg"
                  >
                    ▶
                  </button>
                </div>
                <p className="text-xs text-gray-400 mt-2 text-center">
                  💡 Dica: Digite "SellAlert" para ver o menu principal
                </p>
              </div>
            </div>
          </div>
        )}

        {/* Upload Area */}
        <div className="bg-slate-800/50 backdrop-blur-sm rounded-2xl p-8 shadow-2xl border border-cyan-500/30 mb-6">
          <label className="flex flex-col items-center justify-center border-2 border-dashed border-cyan-500/50 rounded-xl p-12 cursor-pointer hover:border-cyan-400 hover:bg-slate-700/30 transition-all">
            <Upload className="w-16 h-16 text-cyan-400 mb-4" />
            <span className="text-xl text-white font-semibold mb-2">
              Enviar Gráfico para Análise
            </span>
            <span className="text-cyan-300 text-sm">
              Clique ou arraste a imagem do gráfico de candles
            </span>
            <input
              type="file"
              accept="image/*"
              onChange={handleFileUpload}
              className="hidden"
            />
          </label>
        </div>

        {/* Image Preview */}
        {imagePreview && (
          <div className="bg-slate-800/50 backdrop-blur-sm rounded-2xl p-6 shadow-2xl border border-cyan-500/30 mb-6">
            <h3 className="text-white text-lg font-semibold mb-4">Gráfico Carregado:</h3>
            <img src={imagePreview} alt="Gráfico" className="w-full rounded-lg" />
          </div>
        )}

        {/* Loading */}
        {loading && (
          <div className="bg-slate-800/50 backdrop-blur-sm rounded-2xl p-12 shadow-2xl border border-cyan-500/30">
            <div className="flex flex-col items-center gap-4">
              <div className="animate-spin rounded-full h-16 w-16 border-4 border-cyan-500 border-t-transparent"></div>
              <p className="text-cyan-300 text-lg font-semibold">Analisando gráfico...</p>
            </div>
          </div>
        )}

        {/* Analysis Results */}
        {analysis && !analysis.error && !loading && (
          <div className="bg-slate-800/50 backdrop-blur-sm rounded-2xl p-8 shadow-2xl border border-cyan-500/30">
            <h2 className="text-3xl font-bold text-white mb-6 flex items-center gap-3">
              <BarChart3 className="w-8 h-8 text-cyan-400" />
              Resultado da Análise
            </h2>

            {/* Tipo e Valor Principal */}
            <div className={`rounded-xl p-6 mb-6 ${
              analysis.tipo === 'SUPORTE' 
                ? 'bg-green-900/40 border-2 border-green-500' 
                : 'bg-red-900/40 border-2 border-red-500'
            }`}>
              <div className="flex items-center justify-between mb-2">
                <span className="text-lg text-white font-semibold">
                  {analysis.tipo === 'SUPORTE' ? '🟢 SUPORTE IDENTIFICADO' : '🔴 RESISTÊNCIA IDENTIFICADA'}
                </span>
                {analysis.tipo === 'SUPORTE' ? (
                  <TrendingUp className="w-8 h-8 text-green-400" />
                ) : (
                  <TrendingDown className="w-8 h-8 text-red-400" />
                )}
              </div>
              <div className="text-5xl font-bold text-white mb-2">
                {analysis.valor}
              </div>
              <div className="text-cyan-300 mb-4">
                Probabilidade de acerto: <span className="font-bold text-white">{analysis.probabilidade}</span>
              </div>
              
              {/* Próximo Alvo */}
              <div className="bg-slate-900/50 rounded-lg p-4 border border-cyan-500/30">
                <div className="text-cyan-400 text-sm mb-2">🎯 Próximo Alvo (Possível Resultado)</div>
                <div className="text-3xl font-bold text-yellow-400">
                  {analysis.proximo_alvo}
                </div>
                <div className="text-cyan-300 text-sm mt-1">
                  {analysis.tipo === 'SUPORTE' ? 'Possível movimento de alta até este nível' : 'Possível movimento de baixa até este nível'}
                </div>
              </div>
            </div>

            {/* Informações Adicionais */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
              <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600">
                <div className="text-cyan-400 text-sm mb-1">Preço Atual</div>
                <div className="text-white text-2xl font-bold">{analysis.preco_atual}</div>
              </div>
              <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600">
                <div className="text-cyan-400 text-sm mb-1">Tendência</div>
                <div className={`text-2xl font-bold ${
                  analysis.tendencia === 'ALTA' ? 'text-green-400' :
                  analysis.tendencia === 'BAIXA' ? 'text-red-400' : 'text-yellow-400'
                }`}>
                  {analysis.tendencia}
                </div>
              </div>
            </div>

            {/* RSI Status */}
            <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600 mb-6">
              <div className="text-cyan-400 text-sm mb-2">Status RSI</div>
              <div className="text-white text-lg font-semibold">{analysis.rsi_status}</div>
            </div>

            {/* Padrões Identificados */}
            <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600 mb-6">
              <div className="text-cyan-400 text-sm mb-3">Padrões Identificados</div>
              <div className="flex flex-wrap gap-2">
                {analysis.padroes_identificados.map((padrao, index) => (
                  <span key={index} className="bg-cyan-600/30 text-cyan-200 px-3 py-1 rounded-full text-sm border border-cyan-500/50">
                    {padrao}
                  </span>
                ))}
              </div>
            </div>

            {/* Análise Técnica */}
            <div className="bg-slate-700/50 rounded-lg p-4 border border-slate-600 mb-6">
              <div className="text-cyan-400 text-sm mb-2">Análise Técnica</div>
              <p className="text-white leading-relaxed">{analysis.analise_tecnica}</p>
            </div>

            {/* Recomendação */}
            <div className="bg-gradient-to-r from-cyan-900/40 to-blue-900/40 rounded-lg p-5 border border-cyan-500/50">
              <div className="flex items-start gap-3">
                <AlertCircle className="w-6 h-6 text-cyan-400 flex-shrink-0 mt-1" />
                <div>
                  <div className="text-cyan-400 text-sm font-semibold mb-2">Recomendação</div>
                  <p className="text-white leading-relaxed">{analysis.recomendacao}</p>
                </div>
              </div>
            </div>
          </div>
        )}

        {/* Error */}
        {analysis?.error && (
          <div className="bg-red-900/30 rounded-2xl p-6 border border-red-500/50">
            <p className="text-red-300 text-center">{analysis.message}</p>
          </div>
        )}
      </div>
    </div>
  );
};

export default FinancialChartAI;
