<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ultra Productivity - Alarmas</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        body { background-color: #050505; color: white; font-family: sans-serif; }
        .glass-card {
            background: rgba(15, 15, 15, 0.7);
            backdrop-filter: blur(16px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 1.5rem;
        }
        .neon-border { border-color: rgba(147, 51, 234, 0.5); box-shadow: 0 0 15px rgba(147, 51, 234, 0.2); }
        @keyframes pulse-neon { 0% { opacity: 0.5; } 50% { opacity: 1; } 100% { opacity: 0.5; } }
        .alarm-active { animation: pulse-neon 1s infinite; background: rgba(147, 51, 234, 0.2); }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useMemo, useRef } = React;

        const Icon = ({ name, size = 20, className = "" }) => {
            useEffect(() => { if (window.lucide) window.lucide.createIcons(); }, [name]);
            return <i data-lucide={name} style={{ width: size, height: size }} className={className}></i>;
        };

        function App() {
            const [selectedDate, setSelectedDate] = useState(new Date());
            const [tasks, setTasks] = useState(() => {
                const saved = localStorage.getItem('ultra_tasks_v2');
                return saved ? JSON.parse(saved) : {};
            });
            const [newTask, setNewTask] = useState({ title: '', duration: 30, hour: '09:00' });
            const [notificationsEnabled, setNotificationsEnabled] = useState(false);
            
            // Audio para la alarma (usando un sintetizador básico o tono online)
            const audioRef = useRef(new Audio('https://assets.mixkit.co/active_storage/sfx/1016/1016-preview.mp3'));

            useEffect(() => {
                localStorage.setItem('ultra_tasks_v2', JSON.stringify(tasks));
                if (window.lucide) window.lucide.createIcons();
            }, [tasks, selectedDate]);

            // Sistema de chequeo de alarmas cada minuto
            useEffect(() => {
                const interval = setInterval(() => {
                    const now = new Date();
                    const currentHour = now.getHours().toString().padStart(2, '0') + ":" + now.getMinutes().toString().padStart(2, '0');
                    const todayKey = now.toISOString().split('T')[0];
                    
                    if (tasks[todayKey]) {
                        const taskToNotify = tasks[todayKey].find(t => t.hour === currentHour && !t.notified && !t.completed);
                        if (taskToNotify) {
                            triggerAlarm(taskToNotify);
                        }
                    }
                }, 10000); // Check cada 10 seg
                return () => clearInterval(interval);
            }, [tasks]);

            const requestPermission = () => {
                Notification.requestPermission().then(permission => {
                    if (permission === "granted") setNotificationsEnabled(true);
                });
            };

            const triggerAlarm = (task) => {
                // Notificación visual
                if (Notification.permission === "granted") {
                    new Notification("¡Hora de tu tarea!", {
                        body: `${task.title} (${task.duration} min)`,
                        icon: "https://cdn-icons-png.flaticon.com/512/1827/1827347.png"
                    });
                }
                
                // Sonido
                audioRef.current.play().catch(e => console.log("Interacción requerida para sonido"));
                
                // Marcar como notificada para no repetir
                const todayKey = new Date().toISOString().split('T')[0];
                const updated = tasks[todayKey].map(t => t.id === task.id ? {...t, notified: true} : t);
                setTasks({...tasks, [todayKey]: updated});
            };

            const dateKey = selectedDate.toISOString().split('T')[0];
            const currentDayTasks = tasks[dateKey] || [];

            const addTask = (e) => {
                e.preventDefault();
                if (!newTask.title.trim()) return;
                const updatedTasks = {
                    ...tasks,
                    [dateKey]: [...currentDayTasks, { ...newTask, id: Date.now(), completed: false, notified: false }]
                        .sort((a, b) => a.hour.localeCompare(b.hour))
                };
                setTasks(updatedTasks);
                setNewTask({ ...newTask, title: '' });
            };

            const toggleTask = (taskId) => {
                const updated = currentDayTasks.map(t => t.id === taskId ? { ...t, completed: !t.completed } : t);
                setTasks({ ...tasks, [dateKey]: updated });
            };

            const deleteTask = (taskId) => {
                const updated = currentDayTasks.filter(t => t.id !== taskId);
                setTasks({ ...tasks, [dateKey]: updated });
            };

            return (
                <div className="min-h-screen p-4 md:p-8">
                    <div className="max-w-xl mx-auto space-y-6">
                        
                        <header className="text-center space-y-4">
                            <h1 className="text-4xl font-black bg-gradient-to-r from-purple-400 to-pink-500 bg-clip-text text-transparent">
                                FOCUS ALARM
                            </h1>
                            {!notificationsEnabled && (
                                <button 
                                    onClick={requestPermission}
                                    className="text-xs bg-purple-500/20 text-purple-300 px-4 py-2 rounded-full border border-purple-500/30 animate-pulse"
                                >
                                    🔔 Toca aquí para activar alertas en el móvil
                                </button>
                            )}
                        </header>

                        <div className="flex justify-between items-center glass-card p-4">
                            <button onClick={() => {
                                const d = new Date(selectedDate); d.setDate(d.getDate() - 1); setSelectedDate(d);
                            }}><Icon name="chevron-left" /></button>
                            <span className="font-bold capitalize">
                                {selectedDate.toLocaleDateString('es-ES', { weekday: 'short', day: 'numeric', month: 'short' })}
                            </span>
                            <button onClick={() => {
                                const d = new Date(selectedDate); d.setDate(d.getDate() + 1); setSelectedDate(d);
                            }}><Icon name="chevron-right" /></button>
                        </div>

                        <GlassCard className="p-6 neon-border">
                            <form onSubmit={addTask} className="space-y-4">
                                <input 
                                    type="text" 
                                    placeholder="Tarea..."
                                    className="w-full bg-white/5 border border-white/10 rounded-xl p-3 focus:outline-none focus:border-purple-500"
                                    value={newTask.title}
                                    onChange={(e) => setNewTask({...newTask, title: e.target.value})}
                                />
                                <div className="flex gap-2">
                                    <input 
                                        type="time" 
                                        className="flex-1 bg-white/5 border border-white/10 rounded-xl p-3"
                                        value={newTask.hour}
                                        onChange={(e) => setNewTask({...newTask, hour: e.target.value})}
                                    />
                                    <button className="bg-purple-600 px-6 rounded-xl font-bold">
                                        <Icon name="plus" />
                                    </button>
                                </div>
                            </form>
                        </GlassCard>

                        <div className="space-y-3">
                            {currentDayTasks.map(task => (
                                <div key={task.id} className={`glass-card p-4 flex items-center justify-between transition-all ${task.completed ? 'opacity-40' : 'border-l-4 border-purple-500'}`}>
                                    <div className="flex items-center gap-4">
                                        <button onClick={() => toggleTask(task.id)}>
                                            {task.completed ? <Icon name="check-circle" className="text-green-400"/> : <Icon name="circle"/>}
                                        </button>
                                        <div>
                                            <p className="font-bold">{task.hour}</p>
                                            <p className="text-sm text-gray-300">{task.title}</p>
                                        </div>
                                    </div>
                                    <button onClick={() => deleteTask(task.id)} className="text-gray-500"><Icon name="trash-2" size={16}/></button>
                                </div>
                            ))}
                        </div>
                    </div>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
