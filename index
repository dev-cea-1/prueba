import React, { useState, useEffect, useRef } from 'react';
import { 
  Trophy, Users, Gift, Settings, Trash2, Download, Upload, Play, 
  Shuffle, ArrowRight, UserPlus, Dice5, Lock, X, Brain, Clock, 
  CheckCircle, BarChart2, Save, Edit3, Loader, LogOut, Plus, Star 
} from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { 
  getFirestore, collection, addDoc, onSnapshot, doc, updateDoc, 
  deleteDoc, writeBatch, setDoc, increment, serverTimestamp 
} from 'firebase/firestore';

// --- CONFIGURACIÓN FIREBASE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// Paths de Firestore (CORREGIDOS)
// Estructura: artifacts/{appId}/public/data (Documento base, 4 segmentos)
const PATH_PUBLIC_DOC = `artifacts/${appId}/public/data`;

// Para documentos individuales, necesitamos 6 segmentos (par): .../data/{collection}/{doc}
// Usamos una subcolección 'system' para guardar configuraciones únicas
const REF_GAME_STATE = doc(db, PATH_PUBLIC_DOC, 'system', 'gameState');
const REF_QUESTIONS = doc(db, PATH_PUBLIC_DOC, 'system', 'questionsList');

// Para colecciones (listas), necesitamos 5 segmentos (impar): .../data/{collection}
const REF_PARTICIPANTS_COL = collection(db, PATH_PUBLIC_DOC, 'participants');


// --- UTILERÍAS ---
const pickRandomWinner = (participants) => {
  if (participants.length === 0) return null;
  const randomIndex = Math.floor(Math.random() * participants.length);
  return participants[randomIndex];
};

// --- COMPONENTES UI ---

const Confetti = () => {
  const [particles, setParticles] = useState([]);
  useEffect(() => {
    const colors = ['#FFC700', '#FF0000', '#2E3192', '#41BBC7', '#8F00FF'];
    const newParticles = Array.from({ length: 50 }).map((_, i) => ({
      id: i,
      x: Math.random() * 100,
      y: -10 - Math.random() * 20,
      color: colors[Math.floor(Math.random() * colors.length)],
      size: Math.random() * 10 + 5,
      rotation: Math.random() * 360,
      speed: Math.random() * 1 + 0.5,
      delay: Math.random() * 2
    }));
    setParticles(newParticles);
  }, []);

  return (
    <div className="fixed inset-0 pointer-events-none z-50 overflow-hidden">
      {particles.map((p) => (
        <div key={p.id} className="absolute animate-fall"
          style={{
            left: `${p.x}%`, top: `${p.y}%`, width: `${p.size}px`, height: `${p.size}px`,
            backgroundColor: p.color, transform: `rotate(${p.rotation}deg)`,
            animation: `fall ${3 + p.speed}s linear infinite`, animationDelay: `${p.delay}s`
          }}
        />
      ))}
      <style>{`@keyframes fall { 0% { transform: translateY(0) rotate(0deg); opacity: 1; } 100% { transform: translateY(100vh) rotate(360deg); opacity: 0; } }`}</style>
    </div>
  );
};

// --- VISTA DEL JUGADOR (CLIENTE) ---
// Esta vista se activa cuando el usuario ya está registrado
const PlayerView = ({ participantId, participantData }) => {
  const [gameState, setGameState] = useState(null);
  const [hasAnswered, setHasAnswered] = useState(false);
  const [lastScore, setLastScore] = useState(0);

  useEffect(() => {
    // Escuchar estado del juego
    const unsub = onSnapshot(REF_GAME_STATE, (doc) => {
      if (doc.exists()) {
        const data = doc.data();
        setGameState(data);
        // Resetear estado de respuesta si cambia la pregunta ID
        const localAnswered = localStorage.getItem(`answered_${data.currentQuestionId}`);
        setHasAnswered(!!localAnswered);
      }
    });
    return () => unsub();
  }, []);

  const handleAnswer = async (index) => {
    if (hasAnswered || !gameState || gameState.status !== 'active') return;

    setHasAnswered(true);
    localStorage.setItem(`answered_${gameState.currentQuestionId}`, 'true');

    // Cálculo de puntaje (Kahoot style): Base 1000 - tiempo transcurrido
    let score = 0;
    if (index === gameState.correctIndex) {
      const now = Date.now();
      const startTime = gameState.startTime?.toMillis ? gameState.startTime.toMillis() : now;
      const elapsedSec = (now - startTime) / 1000;
      // Puntaje max 1000, mínimo 500 si respondes bien, decae en 10 seg
      score = Math.max(500, Math.round(1000 - (elapsedSec * 50))); 
    }

    setLastScore(score);

    // Actualizar puntaje del usuario en Firestore
    try {
      const pRef = doc(db, PATH_PUBLIC_DOC, 'participants', participantId);
      await updateDoc(pRef, {
        score: increment(score),
        lastAnswerCorrect: score > 0
      });
    } catch (e) {
      console.error("Error enviando respuesta", e);
    }
  };

  if (!gameState) return <div className="flex h-screen items-center justify-center text-white bg-slate-900">Cargando evento...</div>;

  // 1. Pantalla de Espera (Lobby)
  if (gameState.status === 'lobby') {
    return (
      <div className="flex flex-col h-screen bg-slate-900 p-6 text-center items-center justify-center">
        <div className="animate-bounce mb-6"><Trophy className="w-20 h-20 text-yellow-400" /></div>
        <h1 className="text-3xl font-bold text-white mb-2">¡Estás dentro!</h1>
        <p className="text-xl text-indigo-300 font-mono mb-8">{participantData?.name}</p>
        <div className="bg-slate-800 p-4 rounded-xl border border-slate-700 w-full max-w-sm">
          <p className="text-gray-400 text-sm mb-1">Tu Puntaje Actual</p>
          <p className="text-4xl font-black text-white">{participantData?.score || 0}</p>
        </div>
        <p className="mt-12 text-gray-500 animate-pulse">Esperando al presentador...</p>
      </div>
    );
  }

  // 2. Pantalla de Pregunta Activa
  if (gameState.status === 'active') {
    const colors = ["bg-red-500", "bg-blue-500", "bg-yellow-500", "bg-green-500"];
    const shapes = ["▲", "◆", "●", "■"];
    
    return (
      <div className="flex flex-col h-screen bg-slate-900">
        <div className="p-4 bg-slate-800 border-b border-slate-700">
          <h2 className="text-white font-bold text-lg text-center">{gameState.currentQuestionText}</h2>
        </div>
        
        {hasAnswered ? (
          <div className="flex-1 flex flex-col items-center justify-center p-8 text-center">
            <div className="bg-slate-800/50 p-8 rounded-full mb-4">
              <Clock className="w-12 h-12 text-indigo-400 animate-pulse" />
            </div>
            <h3 className="text-2xl text-white font-bold">¡Respuesta enviada!</h3>
            <p className="text-gray-400">Espera a ver si acertaste...</p>
          </div>
        ) : (
          <div className="flex-1 grid grid-cols-2 gap-4 p-4 content-center">
            {gameState.options && gameState.options.map((opt, idx) => (
              <button
                key={idx}
                onClick={() => handleAnswer(idx)}
                className={`${colors[idx]} rounded-xl p-4 flex flex-col items-center justify-center h-40 shadow-lg active:scale-95 transition-transform`}
              >
                <span className="text-4xl text-white/40 mb-2">{shapes[idx]}</span>
                <span className="text-white font-bold text-lg leading-tight">{opt}</span>
              </button>
            ))}
          </div>
        )}
      </div>
    );
  }

  // 3. Pantalla de Resultados (Reveal)
  if (gameState.status === 'reveal') {
    return (
      <div className="flex flex-col h-screen bg-slate-900 items-center justify-center p-6 text-center">
        {lastScore > 0 ? (
          <>
            <Confetti />
            <CheckCircle className="w-24 h-24 text-green-500 mb-6" />
            <h2 className="text-4xl font-bold text-white mb-2">¡Correcto!</h2>
            <p className="text-2xl text-green-400 font-mono">+{lastScore} pts</p>
          </>
        ) : (
          <>
            <X className="w-24 h-24 text-red-500 mb-6" />
            <h2 className="text-3xl font-bold text-white mb-2">Incorrecto</h2>
            <p className="text-gray-400">¡Más suerte en la próxima!</p>
          </>
        )}
        <div className="mt-12 bg-slate-800 p-4 rounded-xl w-full max-w-sm">
          <p className="text-gray-400 text-sm">Puntaje Total</p>
          <p className="text-3xl font-bold text-white">{participantData?.score || 0}</p>
        </div>
      </div>
    );
  }
  
  return null;
};

// --- APP PRINCIPAL (CONTENEDOR) ---

export default function App() {
  const [user, setUser] = useState(null);
  const [isAdmin, setIsAdmin] = useState(false);
  
  // Estado Local del Usuario
  const [localParticipantId, setLocalParticipantId] = useState(localStorage.getItem('myParticipantId'));
  const [myParticipantData, setMyParticipantData] = useState(null);
  
  // Estados de Admin
  const [view, setView] = useState('landing'); // landing, admin_dashboard, admin_game
  const [adminPin, setAdminPin] = useState('');
  const [showAdminLogin, setShowAdminLogin] = useState(false);
  const [participants, setParticipants] = useState([]);
  
  // Trivia Management
  const [questionsList, setQuestionsList] = useState([]);
  const [currentQIndex, setCurrentQIndex] = useState(0);
  const [gameState, setGameState] = useState({ status: 'lobby' }); // Local copy of DB state

  // Configuración de nueva pregunta
  const [newQuestion, setNewQuestion] = useState({ 
    text: '', options: ['', '', '', ''], correct: 0, points: 1000 
  });
  const [showQModal, setShowQModal] = useState(false);

  // --- EFECTOS ---

  // 1. Autenticación Anónima Inicial
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, setUser);
  }, []);

  // 2. Escuchar Participantes (Solo si es Admin)
  useEffect(() => {
    if (!user || !isAdmin) return;
    // Escuchamos REF_PARTICIPANTS_COL (5 segmentos)
    return onSnapshot(REF_PARTICIPANTS_COL, (snap) => {
      const data = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      // Ordenar por puntaje descendente
      data.sort((a, b) => (b.score || 0) - (a.score || 0));
      setParticipants(data);
    });
  }, [user, isAdmin]);

  // 3. Escuchar Mis Datos (Si soy participante)
  useEffect(() => {
    if (!user || !localParticipantId) return;
    const ref = doc(db, PATH_PUBLIC_DOC, 'participants', localParticipantId);
    return onSnapshot(ref, (snap) => {
      if (snap.exists()) setMyParticipantData(snap.data());
    });
  }, [user, localParticipantId]);

  // 4. Cargar Preguntas (Solo Admin)
  useEffect(() => {
    if (!user || !isAdmin) return;
    return onSnapshot(REF_QUESTIONS, (snap) => {
      if (snap.exists()) setQuestionsList(snap.data().list || []);
    });
  }, [user, isAdmin]);

  // 5. Escuchar Game State (Admin también necesita saber estado actual)
  useEffect(() => {
    if (!user) return;
    return onSnapshot(REF_GAME_STATE, (snap) => {
      if (snap.exists()) setGameState(snap.data());
      else {
        // Init game state if not exists
        setDoc(REF_GAME_STATE, { status: 'lobby', currentQuestionId: null });
      }
    });
  }, [user]);

  // --- FUNCIONES DE REGISTRO (USUARIO) ---
  const [regName, setRegName] = useState('');
  const [regHandle, setRegHandle] = useState('');

  const handleRegister = async (e) => {
    e.preventDefault();
    if (!regName.trim()) return;
    
    try {
      // Usamos la colección de participantes (5 segmentos)
      const docRef = await addDoc(REF_PARTICIPANTS_COL, {
        name: regName,
        handle: regHandle || '@anonimo',
        score: 0,
        timestamp: serverTimestamp(),
        likes: 0, shares: 0, comments: 0 // Marketing fields
      });
      localStorage.setItem('myParticipantId', docRef.id);
      setLocalParticipantId(docRef.id);
    } catch (err) {
      console.error(err);
      alert("Error al registrar. Revisa tu conexión.");
    }
  };

  // --- FUNCIONES DE ADMIN ---

  const handleAdminLogin = () => {
    if (adminPin === '0000') {
      setIsAdmin(true);
      setShowAdminLogin(false);
      setView('admin_dashboard');
    } else {
      alert("PIN Incorrecto");
    }
  };

  const saveQuestion = async () => {
    const updatedList = [...questionsList, { ...newQuestion, id: Date.now() }];
    await setDoc(REF_QUESTIONS, { list: updatedList });
    setNewQuestion({ text: '', options: ['', '', '', ''], correct: 0, points: 1000 });
    setShowQModal(false);
  };

  const deleteQuestion = async (idx) => {
    const updatedList = questionsList.filter((_, i) => i !== idx);
    await setDoc(REF_QUESTIONS, { list: updatedList });
  };

  // --- LOGICA DEL JUEGO (TRIVIA) ---

  const startGame = async () => {
    if (questionsList.length === 0) return alert("¡Agrega preguntas primero!");
    setCurrentQIndex(0);
    setView('admin_game');
    // Resetear estado a lobby por seguridad al entrar
    await updateDoc(REF_GAME_STATE, { status: 'lobby' });
  };

  const launchQuestion = async () => {
    const q = questionsList[currentQIndex];
    if (!q) return;

    // Actualizar estado global para que todos los teléfonos cambien
    await updateDoc(REF_GAME_STATE, {
      status: 'active',
      currentQuestionId: q.id,
      currentQuestionText: q.text,
      options: q.options,
      correctIndex: q.correct, // Enviamos el índice correcto (en una app real seria hidden)
      startTime: serverTimestamp()
    });
  };

  const revealAnswer = async () => {
    await updateDoc(REF_GAME_STATE, { status: 'reveal' });
  };

  const nextQuestion = async () => {
    if (currentQIndex + 1 < questionsList.length) {
      setCurrentQIndex(prev => prev + 1);
      await updateDoc(REF_GAME_STATE, { status: 'lobby' }); // Pausa breve entre preguntas
    } else {
      alert("Fin de la trivia");
      await updateDoc(REF_GAME_STATE, { status: 'lobby' });
      setView('admin_dashboard');
    }
  };

  // --- VISTA 1: LANDING / LOGIN / REGISTRO ---

  if (view === 'landing') {
    // Si ya soy participante registrado, voy directo a PlayerView
    if (localParticipantId) {
      return <PlayerView participantId={localParticipantId} participantData={myParticipantData} />;
    }

    return (
      <div className="min-h-screen bg-gradient-to-br from-indigo-900 via-purple-900 to-pink-900 flex flex-col items-center justify-center p-4">
        <div className="bg-white/10 backdrop-blur-md p-8 rounded-2xl shadow-2xl max-w-md w-full border border-white/20">
          
          {/* Header */}
          <div className="text-center mb-8">
            <div className="flex justify-center mb-4"><Trophy className="w-16 h-16 text-yellow-400" /></div>
            <h1 className="text-4xl font-extrabold text-white mb-2">Sorteos Social</h1>
            <p className="text-indigo-200">Participa y Gana en Vivo</p>
          </div>

          {/* Formulario de Registro de Usuario */}
          {!showAdminLogin ? (
            <div className="space-y-4 animate-fadeIn">
              <input 
                className="w-full bg-white/20 border border-white/30 rounded-xl p-4 text-white placeholder-indigo-200 focus:outline-none focus:ring-2 focus:ring-pink-500"
                placeholder="Tu Nombre Completo"
                value={regName}
                onChange={e => setRegName(e.target.value)}
              />
              <input 
                className="w-full bg-white/20 border border-white/30 rounded-xl p-4 text-white placeholder-indigo-200 focus:outline-none focus:ring-2 focus:ring-pink-500"
                placeholder="@Usuario (Instagram/TikTok)"
                value={regHandle}
                onChange={e => setRegHandle(e.target.value)}
              />
              <button 
                onClick={handleRegister}
                disabled={!user}
                className="w-full bg-gradient-to-r from-pink-500 to-purple-600 text-white font-bold py-4 rounded-xl shadow-lg transform active:scale-95 transition-transform hover:brightness-110 flex items-center justify-center gap-2"
              >
                {!user ? <Loader className="animate-spin"/> : <UserPlus />} 
                ¡Entrar al Juego!
              </button>
              
              <div className="pt-6 mt-6 border-t border-white/10 text-center">
                <button onClick={() => setShowAdminLogin(true)} className="text-xs text-indigo-300 hover:text-white flex items-center justify-center gap-1 mx-auto">
                  <Lock className="w-3 h-3" /> Acceso Organizador
                </button>
              </div>
            </div>
          ) : (
            // Login Admin
            <div className="space-y-4 animate-fadeIn">
              <div className="flex justify-between items-center text-white mb-2">
                <h3 className="font-bold">Acceso Admin</h3>
                <button onClick={() => setShowAdminLogin(false)}><X className="w-5 h-5" /></button>
              </div>
              <input 
                type="password" maxLength="4" placeholder="PIN (0000)"
                className="w-full text-center text-3xl tracking-widest bg-white text-black rounded-xl p-4 font-bold"
                value={adminPin} onChange={e => setAdminPin(e.target.value)}
              />
              <button onClick={handleAdminLogin} className="w-full bg-black text-white font-bold py-4 rounded-xl">
                Entrar
              </button>
            </div>
          )}
        </div>
      </div>
    );
  }

  // --- VISTA 2: DASHBOARD ADMIN ---
  
  if (view === 'admin_dashboard') {
    return (
      <div className="min-h-screen bg-gray-50 flex flex-col">
        {/* Navbar */}
        <header className="bg-white border-b px-6 py-4 flex justify-between items-center sticky top-0 z-20">
          <div className="flex items-center gap-2">
            <div className="bg-indigo-600 p-2 rounded-lg"><Settings className="w-5 h-5 text-white" /></div>
            <h2 className="text-xl font-bold text-gray-800">Panel de Control</h2>
          </div>
          <div className="flex gap-3">
             <div className="bg-green-100 text-green-700 px-3 py-1 rounded-full text-sm font-bold flex items-center gap-2">
               <Users className="w-4 h-4"/> {participants.length} Activos
             </div>
             <button onClick={() => { setIsAdmin(false); setView('landing'); }} className="text-gray-500 hover:text-red-500"><LogOut className="w-5 h-5" /></button>
          </div>
        </header>

        <main className="flex-1 p-6 max-w-6xl w-full mx-auto grid grid-cols-1 lg:grid-cols-2 gap-8">
          
          {/* COLUMNA IZQUIERDA: Gestión de Trivia */}
          <section className="space-y-6">
            <div className="bg-white rounded-2xl shadow-sm border border-gray-100 p-6">
              <div className="flex justify-between items-center mb-6">
                <h3 className="text-lg font-bold flex items-center gap-2 text-gray-800"><Brain className="w-5 h-5 text-purple-600"/> Banco de Preguntas ({questionsList.length})</h3>
                <button onClick={() => setShowQModal(true)} className="bg-purple-600 text-white px-4 py-2 rounded-lg text-sm font-bold hover:bg-purple-700 flex items-center gap-2"><Plus className="w-4 h-4"/> Nueva</button>
              </div>

              <div className="space-y-3 max-h-[500px] overflow-y-auto pr-2">
                {questionsList.length === 0 && <p className="text-gray-400 text-center py-8">No hay preguntas configuradas.</p>}
                {questionsList.map((q, idx) => (
                  <div key={q.id} className="bg-gray-50 p-4 rounded-xl border border-gray-200 relative group">
                    <div className="flex justify-between items-start mb-2">
                      <span className="bg-gray-200 text-gray-600 text-xs font-bold px-2 py-1 rounded">Q{idx+1}</span>
                      <button onClick={() => deleteQuestion(idx)} className="text-gray-400 hover:text-red-500"><Trash2 className="w-4 h-4"/></button>
                    </div>
                    <p className="font-medium text-gray-800 mb-2">{q.text}</p>
                    <div className="grid grid-cols-2 gap-2 text-xs text-gray-500">
                      {q.options.map((opt, i) => (
                        <div key={i} className={`px-2 py-1 rounded border ${i === q.correct ? 'bg-green-100 border-green-200 text-green-700 font-bold' : 'border-transparent'}`}>
                          {opt}
                        </div>
                      ))}
                    </div>
                  </div>
                ))}
              </div>

              <div className="mt-6 pt-6 border-t border-gray-100">
                <button 
                  onClick={startGame}
                  className="w-full bg-gradient-to-r from-purple-600 to-indigo-600 text-white py-4 rounded-xl font-bold text-lg shadow-lg hover:brightness-110 flex items-center justify-center gap-2"
                >
                  <Play className="w-6 h-6" /> INICIAR MODO LIVE
                </button>
                <p className="text-center text-xs text-gray-400 mt-2">Esto cambiará la pantalla de todos los usuarios conectados.</p>
              </div>
            </div>
          </section>

          {/* COLUMNA DERECHA: Ranking en Vivo & Datos */}
          <section className="space-y-6">
             <div className="bg-white rounded-2xl shadow-sm border border-gray-100 p-6 h-full flex flex-col">
               <h3 className="text-lg font-bold flex items-center gap-2 text-gray-800 mb-6"><Trophy className="w-5 h-5 text-yellow-500"/> Tabla de Líderes</h3>
               
               <div className="flex-1 overflow-y-auto">
                 <table className="w-full text-left text-sm">
                   <thead className="bg-gray-50 text-gray-500">
                     <tr>
                       <th className="p-3 rounded-l-lg">#</th>
                       <th className="p-3">Participante</th>
                       <th className="p-3 text-right rounded-r-lg">Puntaje</th>
                     </tr>
                   </thead>
                   <tbody className="divide-y divide-gray-100">
                     {participants.slice(0, 50).map((p, i) => (
                       <tr key={p.id} className="hover:bg-gray-50 group">
                         <td className="p-3 font-mono text-gray-400">{i+1}</td>
                         <td className="p-3">
                           <div className="font-bold text-gray-800">{p.name}</div>
                           <div className="text-xs text-gray-400">{p.handle}</div>
                         </td>
                         <td className="p-3 text-right font-bold text-indigo-600">{p.score}</td>
                       </tr>
                     ))}
                   </tbody>
                 </table>
               </div>
               
               <div className="mt-4 flex justify-end">
                  <button className="text-indigo-600 text-sm font-medium hover:underline flex items-center gap-1">
                    <Download className="w-4 h-4"/> Exportar CSV Completo
                  </button>
               </div>
             </div>
          </section>

        </main>

        {/* Modal Nueva Pregunta */}
        {showQModal && (
          <div className="fixed inset-0 bg-black/50 backdrop-blur-sm z-50 flex items-center justify-center p-4">
            <div className="bg-white rounded-2xl w-full max-w-lg p-6 shadow-2xl">
              <h3 className="text-xl font-bold mb-4">Configurar Pregunta</h3>
              <div className="space-y-4">
                <div>
                  <label className="block text-xs font-bold text-gray-500 mb-1">Enunciado</label>
                  <input className="w-full border-2 border-gray-200 rounded-lg p-3" value={newQuestion.text} onChange={e => setNewQuestion({...newQuestion, text: e.target.value})} placeholder="¿Cuál es la capital de...?" />
                </div>
                <div className="grid grid-cols-2 gap-3">
                  {newQuestion.options.map((opt, i) => (
                    <div key={i}>
                      <div className="flex justify-between mb-1">
                        <label className="text-xs font-bold text-gray-400">Opción {i+1}</label>
                        <input type="radio" name="correct" checked={newQuestion.correct === i} onChange={() => setNewQuestion({...newQuestion, correct: i})} className="accent-green-500"/>
                      </div>
                      <input 
                        className={`w-full border rounded p-2 text-sm ${newQuestion.correct === i ? 'border-green-500 bg-green-50' : 'border-gray-200'}`}
                        value={opt}
                        onChange={e => {
                          const opts = [...newQuestion.options];
                          opts[i] = e.target.value;
                          setNewQuestion({...newQuestion, options: opts});
                        }}
                      />
                    </div>
                  ))}
                </div>
                <div className="flex gap-3 mt-6">
                  <button onClick={() => setShowQModal(false)} className="flex-1 py-3 text-gray-500 font-bold bg-gray-100 rounded-xl">Cancelar</button>
                  <button onClick={saveQuestion} className="flex-1 py-3 bg-black text-white font-bold rounded-xl">Guardar</button>
                </div>
              </div>
            </div>
          </div>
        )}
      </div>
    );
  }

  // --- VISTA 3: ADMIN MODO JUEGO LIVE ---
  if (view === 'admin_game') {
    const currentQ = questionsList[currentQIndex];

    return (
      <div className="min-h-screen bg-slate-900 flex flex-col items-center">
        {/* Top Control Bar */}
        <div className="w-full bg-slate-800 p-4 flex justify-between items-center border-b border-slate-700">
          <div className="text-white">
            <span className="text-gray-400 text-sm uppercase tracking-wider">En Vivo</span>
            <div className="font-bold">Pregunta {currentQIndex + 1} / {questionsList.length}</div>
          </div>
          <div className="flex gap-4">
             <div className="bg-slate-900 px-4 py-2 rounded-lg text-white font-mono border border-slate-600">
               Estado: <span className={gameState.status === 'active' ? 'text-green-400' : 'text-yellow-400'}>{gameState.status.toUpperCase()}</span>
             </div>
             <button onClick={() => setView('admin_dashboard')} className="text-gray-400 hover:text-white"><X /></button>
          </div>
        </div>

        {/* Main Game Stage */}
        <div className="flex-1 w-full max-w-5xl p-8 flex flex-col justify-center">
          
          <div className="bg-white rounded-2xl p-8 text-center shadow-2xl mb-8">
            <h2 className="text-3xl md:text-5xl font-black text-slate-800 mb-8">{currentQ.text}</h2>
            
            <div className="grid grid-cols-2 gap-4">
              {currentQ.options.map((opt, i) => (
                <div key={i} className={`p-6 rounded-xl text-xl font-bold text-white transition-all
                  ${gameState.status === 'reveal' && i !== currentQ.correct ? 'opacity-20 bg-gray-400' : ''}
                  ${i === 0 ? 'bg-red-500' : i === 1 ? 'bg-blue-500' : i === 2 ? 'bg-yellow-500' : 'bg-green-500'}
                  ${gameState.status === 'reveal' && i === currentQ.correct ? 'ring-4 ring-offset-4 ring-green-500 scale-105 shadow-xl' : ''}
                `}>
                  {opt}
                </div>
              ))}
            </div>
          </div>

          {/* Controls */}
          <div className="flex justify-center gap-4">
            {gameState.status === 'lobby' && (
              <button onClick={launchQuestion} className="bg-indigo-600 hover:bg-indigo-500 text-white px-8 py-4 rounded-full font-bold text-2xl shadow-lg shadow-indigo-500/50 flex items-center gap-3 transition-all hover:scale-105">
                <Play className="w-8 h-8"/> LANZAR PREGUNTA
              </button>
            )}

            {gameState.status === 'active' && (
              <button onClick={revealAnswer} className="bg-orange-500 hover:bg-orange-400 text-white px-8 py-4 rounded-full font-bold text-2xl shadow-lg flex items-center gap-3 transition-all hover:scale-105">
                <CheckCircle className="w-8 h-8"/> MOSTRAR RESPUESTA
              </button>
            )}

            {gameState.status === 'reveal' && (
              <button onClick={nextQuestion} className="bg-slate-700 hover:bg-slate-600 text-white px-8 py-4 rounded-full font-bold text-2xl shadow-lg flex items-center gap-3 transition-all hover:scale-105">
                SIGUIENTE <ArrowRight className="w-8 h-8"/>
              </button>
            )}
          </div>
          
        </div>
      </div>
    );
  }

  return null;
}
