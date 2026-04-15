<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bamboenyi Audio Workstation</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Babel untuk transpiling JSX di browser -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Google Fonts -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&family=JetBrains+Mono:wght@700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Inter', sans-serif; background-color: #050505; color: #f8fafc; margin: 0; overflow-x: hidden; }
        .font-mono { font-family: 'JetBrains Mono', monospace; }
        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: rgba(255,255,255,0.02); }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: rgba(251,191,36,0.3); border-radius: 10px; }
        
        /* Gaya slider kustom */
        input[type=range] { -webkit-appearance: none; background: transparent; height: 16px; }
        input[type=range]:focus { outline: none; }
        input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; height: 12px; width: 12px; border-radius: 50%; background: #f59e0b; cursor: pointer; border: 2px solid #000; margin-top: -4px; box-shadow: 0 0 8px rgba(245,158,11,0.3); }
        input[type=range]::-webkit-slider-runnable-track { width: 100%; height: 3px; cursor: pointer; background: #1a1a1a; border-radius: 2px; border: 1px solid rgba(255,255,255,0.05); }

        /* Garis Penanda Masa (Playhead) */
        .playhead-line {
            pointer-events: none;
            box-shadow: 0 0 10px #fbbf24, 0 0 3px rgba(251,191,36,0.8);
            z-index: 50;
            width: 2px;
            background-color: #fbbf24;
            height: 100%;
            position: absolute;
            top: 0;
            transition: left 0.1s linear;
        }

        /* Penunjuk segitiga playhead */
        .playhead-marker {
            width: 14px;
            height: 14px;
            background: #fbbf24;
            position: absolute;
            top: -7px;
            left: -6px;
            clip-path: polygon(0% 0%, 100% 0%, 50% 100%);
            z-index: 60;
            filter: drop-shadow(0 0 5px rgba(251,191,36,0.5));
        }

        .waveform-bar {
            transition: height 0.2s ease;
        }

        /* Playlist Active State */
        .playlist-item-active {
            background: rgba(245, 158, 11, 0.1);
            border-color: rgba(245, 158, 11, 0.3);
            color: #f59e0b;
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel" data-type="module">
        import React, { useState, useEffect, useRef, useMemo } from 'https://esm.sh/react@18.2.0';
        import ReactDOM from 'https://esm.sh/react-dom@18.2.0/client';
        import * as Tone from 'https://esm.sh/tone@14.8.49';
        import { 
            Play, Pause, Square, Volume2, Music, Activity, Timer, Trash2, Upload, Headphones, 
            Plus, Loader2, ChevronUp, ChevronDown, Waves, X, Info, RotateCcw, FolderUp, ListMusic, ChevronRight
        } from 'https://esm.sh/lucide-react@0.263.1';

        const App = () => {
            const [isPlaying, setIsPlaying] = useState(false);
            const [isPaused, setIsPaused] = useState(false);
            const [isMetronomeOn, setIsMetronomeOn] = useState(false);
            const [isMetronomeDouble, setIsMetronomeDouble] = useState(true); 
            const [metronomeVolume, setMetronomeVolume] = useState(-12); 
            const [bpm, setBpm] = useState(120);
            const [detectedKey, setDetectedKey] = useState("-");
            const [playbackRate, setPlaybackRate] = useState(1);
            const [transpose, setTranspose] = useState(0); 
            const [currentTime, setCurrentTime] = useState(0);
            const [isAnalyzing, setIsAnalyzing] = useState(false);
            const [projectName, setProjectName] = useState("");
            
            // Playlist States
            const [playlist, setPlaylist] = useState([]); // Array of projects { name, stems, bpm }
            const [currentProjectIndex, setCurrentProjectIndex] = useState(-1);
            const [stems, setStems] = useState([]);

            const players = useRef({}); 
            const volumes = useRef({});
            const solos = useRef({});
            const metronomeEventId = useRef(null);
            const clickSynth = useRef(null);
            const metronomeVolNode = useRef(null);
            const animationRef = useRef(null);

            const masterTrack = useMemo(() => stems.length > 0 ? stems[0] : null, [stems]);

            // --- INIT AUDIO ENGINE ---
            useEffect(() => {
                metronomeVolNode.current = new Tone.Volume(metronomeVolume).toDestination();
                clickSynth.current = new Tone.Synth({
                    oscillator: { type: "sine" },
                    envelope: { attack: 0.001, decay: 0.05, sustain: 0, release: 0.05 }
                }).connect(metronomeVolNode.current);

                const syncPlayhead = () => {
                    if (Tone.Transport.state === 'started') {
                        setCurrentTime(Tone.Transport.seconds);
                    }
                    animationRef.current = requestAnimationFrame(syncPlayhead);
                };
                animationRef.current = requestAnimationFrame(syncPlayhead);

                return () => {
                    Tone.Transport.stop();
                    cancelAnimationFrame(animationRef.current);
                    Object.values(players.current).forEach(p => p.dispose());
                    if (metronomeEventId.current !== null) Tone.Transport.clear(metronomeEventId.current);
                    if (clickSynth.current) clickSynth.current.dispose();
                    if (metronomeVolNode.current) metronomeVolNode.current.dispose();
                };
            }, []);

            // Metronome logic
            useEffect(() => {
                if (metronomeEventId.current !== null) Tone.Transport.clear(metronomeEventId.current);
                const interval = isMetronomeDouble ? "8n" : "4n";
                metronomeEventId.current = Tone.Transport.scheduleRepeat((time) => {
                    const ticks = Tone.Transport.getTicksAtTime(time);
                    const ppq = Tone.Transport.PPQ;
                    const isOnBeat = Math.abs(ticks % ppq) < 10 || Math.abs((ticks % ppq) - ppq) < 10;
                    const isOnOffBeat = Math.abs(ticks % (ppq / 2)) < 10 && !isOnBeat;

                    if (isOnBeat) {
                        const beatIndex = Math.round(ticks / ppq) % 4;
                        if (beatIndex === 0) clickSynth.current.triggerAttackRelease("C5", "32n", time, 1.0);
                        else clickSynth.current.triggerAttackRelease("G4", "32n", time, 0.4);
                    } else if (isMetronomeDouble && isOnOffBeat) {
                        clickSynth.current.triggerAttackRelease("E4", "32n", time, 0.2);
                    }
                }, "8n");
            }, [isMetronomeDouble]);

            // Sync States
            useEffect(() => { 
                const effectiveBpm = bpm * playbackRate;
                if (!isNaN(effectiveBpm) && effectiveBpm > 0) Tone.Transport.bpm.value = effectiveBpm; 
            }, [bpm, playbackRate]);

            useEffect(() => { 
                Object.values(players.current).forEach(p => { if (p) p.playbackRate = playbackRate; }); 
            }, [playbackRate]);

            useEffect(() => { 
                Object.values(players.current).forEach(p => { if (p) p.detune = transpose * 100; }); 
            }, [transpose]);
            
            useEffect(() => {
                if (metronomeVolNode.current) {
                    metronomeVolNode.current.volume.value = metronomeVolume;
                    metronomeVolNode.current.mute = !isMetronomeOn;
                }
            }, [metronomeVolume, isMetronomeOn]);

            // Waveform Generator
            const generateWaveformData = (buffer) => {
                const data = buffer.getChannelData(0);
                const segments = 300; 
                const step = Math.floor(data.length / segments);
                const points = [];
                for (let i = 0; i < segments; i++) {
                    let max = 0;
                    for (let j = 0; j < step; j++) {
                        const val = Math.abs(data[(i * step) + j]);
                        if (val > max) max = val;
                    }
                    points.push(Math.min(1, max * 1.5));
                }
                return points;
            };

            // Tempo Detection
            const detectBPMFromBuffer = (buffer) => {
                const data = buffer.getChannelData(0);
                const sampleRate = buffer.sampleRate;
                const downsample = 256;
                const fluxes = [];
                for (let i = downsample; i < Math.min(data.length, sampleRate * 30); i += downsample) {
                    let flux = 0;
                    for (let j = 0; j < downsample; j++) {
                        const d = Math.abs(data[i + j]) - Math.abs(data[i + j - downsample]);
                        flux += d > 0 ? d : 0;
                    }
                    fluxes.push(flux);
                }
                const minLag = Math.floor((60 / 220) * (sampleRate / downsample)); 
                const maxLag = Math.floor((60 / 80) * (sampleRate / downsample));  
                let bestLag = 0, maxVal = -1;
                for (let lag = minLag; lag < maxLag; lag++) {
                    let sum = 0;
                    for (let i = 0; i < fluxes.length - lag; i++) sum += fluxes[i] * fluxes[i + lag];
                    if (sum > maxVal) { maxVal = sum; bestLag = lag; }
                }
                let result = Math.round(60 / (bestLag * downsample / sampleRate));
                if (result < 95) result *= 2;
                return result;
            };

            const toggleMute = (id) => {
                setStems(prev => prev.map(s => {
                    if (s.id === id) {
                        const newMuted = !s.muted;
                        if (volumes.current[id]) volumes.current[id].mute = newMuted;
                        return { ...s, muted: newMuted };
                    }
                    return s;
                }));
            };

            const toggleSolo = (id) => {
                setStems(prev => {
                    const newStems = prev.map(s => s.id === id ? { ...s, solo: !s.solo } : s);
                    newStems.forEach(s => { if (solos.current[s.id]) solos.current[s.id].solo = s.solo; });
                    return newStems;
                });
            };

            const handleSeek = (e, duration) => {
                if (!duration) return;
                const rect = e.currentTarget.getBoundingClientRect();
                const x = e.clientX - rect.left;
                const percent = Math.max(0, Math.min(1, x / rect.width));
                const seekTime = percent * duration;
                Tone.Transport.seconds = seekTime;
                setCurrentTime(seekTime);
            };

            // --- PLAYLIST LOADING LOGIC ---
            const handlePlaylistLoad = async (event) => {
                const files = Array.from(event.target.files);
                if (files.length === 0) return;

                // Group files by subfolder name (Song Title)
                const projectGroups = {};
                files.forEach(file => {
                    if (!file.type.startsWith('audio/')) return;
                    const pathParts = file.webkitRelativePath.split('/');
                    // Usually: Root/Subfolder/file.mp3
                    const folderName = pathParts.length > 2 ? pathParts[pathParts.length - 2] : "Uncategorized";
                    if (!projectGroups[folderName]) projectGroups[folderName] = [];
                    projectGroups[folderName].push(file);
                });

                const groupNames = Object.keys(projectGroups);
                if (groupNames.length === 0) return;

                setProjectName("Playlist Loaded");
                
                // Construct Playlist Object
                const playlistData = groupNames.map(name => ({
                    name: name,
                    files: projectGroups[name],
                    stems: [],
                    bpm: 120,
                    loaded: false
                }));

                setPlaylist(playlistData);
                // Load the first song automatically
                loadProjectFromPlaylist(0, playlistData);
            };

            const loadProjectFromPlaylist = async (index, currentPlaylist = playlist) => {
                if (index < 0 || index >= currentPlaylist.length) return;
                
                setIsAnalyzing(true);
                setCurrentProjectIndex(index);
                const project = currentPlaylist[index];

                if (Tone.context.state !== 'running') await Tone.start();
                
                // Cleanup Audio
                Tone.Transport.stop();
                Object.values(players.current).forEach(p => p.dispose());
                players.current = {}; volumes.current = {}; solos.current = {};
                setStems([]);
                setCurrentTime(0);

                const newStems = [];
                for (let i = 0; i < project.files.length; i++) {
                    const file = project.files[i];
                    const trackId = `tr-${Date.now()}-${i}`;
                    const buffer = new Tone.ToneAudioBuffer();
                    try {
                        await buffer.load(URL.createObjectURL(file));
                        const vol = new Tone.Volume(0).toDestination();
                        const solo = new Tone.Solo().connect(vol);
                        const player = new Tone.GrainPlayer(buffer).connect(solo);
                        player.playbackRate = playbackRate;
                        player.detune = transpose * 100;
                        player.sync().start(0);
                        
                        players.current[trackId] = player; volumes.current[trackId] = vol; solos.current[trackId] = solo;

                        const waveform = generateWaveformData(buffer);

                        newStems.push({ 
                            id: trackId, 
                            type: i === 0 ? 'full' : 'custom', 
                            label: file.name.split('.')[0], 
                            color: i === 0 ? 'text-amber-400' : 'text-emerald-400', 
                            volume: 0, 
                            muted: false, 
                            solo: false, 
                            fileName: file.name, 
                            loaded: true,
                            duration: buffer.duration,
                            waveform: waveform
                        });

                        if (i === 0) setBpm(detectBPMFromBuffer(buffer));
                    } catch (e) { console.error(e); }
                }
                setStems(newStems);
                setIsAnalyzing(false);
            };

            const handleVolumeChange = (id, val) => {
                const v = parseFloat(val);
                if (volumes.current[id]) volumes.current[id].volume.value = v;
                setStems(prev => prev.map(s => s.id === id ? { ...s, volume: v } : s));
            };

            const formatTime = (s) => `${Math.floor(s / 60)}:${Math.floor(s % 60).toString().padStart(2, '0')}`;

            return (
                <div className="flex h-screen bg-[#050505] text-slate-200 font-sans overflow-hidden">
                    
                    {/* SIDEBAR PLAYLIST */}
                    <aside className="w-64 bg-black/40 border-r border-white/5 flex flex-col shrink-0">
                        <div className="p-4 border-b border-white/5 flex items-center gap-3">
                            <ListMusic className="text-amber-500" size={20} />
                            <h2 className="font-black text-xs uppercase tracking-[0.2em]">Playlist</h2>
                        </div>
                        
                        <div className="flex-1 overflow-y-auto custom-scrollbar p-2 space-y-1">
                            {playlist.length === 0 ? (
                                <div className="py-10 text-center px-4">
                                    <p className="text-[10px] text-slate-600 uppercase font-bold tracking-widest leading-loose italic">
                                        Upload folder yang berisi folder judul lagu
                                    </p>
                                </div>
                            ) : (
                                playlist.map((item, idx) => (
                                    <button 
                                        key={idx}
                                        onClick={() => loadProjectFromPlaylist(idx)}
                                        className={`w-full text-left p-3 rounded-xl border border-transparent transition-all flex items-center justify-between group ${currentProjectIndex === idx ? 'playlist-item-active' : 'hover:bg-white/5'}`}
                                    >
                                        <div className="truncate pr-2">
                                            <p className="text-[11px] font-black uppercase truncate tracking-tighter">{item.name}</p>
                                            <p className="text-[8px] opacity-50 uppercase font-bold">{item.files.length} Tracks</p>
                                        </div>
                                        <ChevronRight size={14} className={`opacity-0 group-hover:opacity-100 transition-opacity ${currentProjectIndex === idx ? 'opacity-100' : ''}`} />
                                    </button>
                                ))
                            )}
                        </div>

                        <div className="p-4 border-t border-white/5">
                            <label className="w-full py-3 bg-white/5 hover:bg-white/10 text-slate-400 rounded-xl text-[10px] font-black uppercase tracking-widest border border-white/5 cursor-pointer transition-all flex items-center justify-center gap-2">
                                <FolderUp size={14} /> Open Playlist
                                <input type="file" webkitdirectory="true" directory="true" className="hidden" onChange={handlePlaylistLoad} />
                            </label>
                        </div>
                    </aside>

                    {/* MAIN WORKSTATION AREA */}
                    <div className="flex-1 flex flex-col relative overflow-hidden">
                        <header className="bg-black/95 backdrop-blur-xl border-b border-white/5 p-2 px-4 sticky top-0 z-[100]">
                            <div className="max-w-7xl mx-auto flex items-center justify-between">
                                <div className="flex items-center gap-2">
                                    <div className="w-8 h-8 bg-amber-500 rounded flex items-center justify-center shadow-lg"><Headphones className="text-black" size={18} /></div>
                                    <div>
                                        <h1 className="text-sm font-black text-white uppercase leading-none tracking-tighter italic">Bamboenyi Audio Workstation</h1>
                                        <p className="text-[6px] text-slate-500 font-bold uppercase tracking-widest">
                                            {currentProjectIndex !== -1 ? playlist[currentProjectIndex].name : "IDLE"}
                                        </p>
                                    </div>
                                </div>
                                <div className="flex items-center gap-1.5 bg-white/[0.02] p-1 rounded-lg border border-white/5 font-bold shadow-inner">
                                    <div className="px-3 py-0.5 border-r border-white/5 text-center"><span className="text-[6px] text-slate-500 block uppercase mb-0.5 tracking-tighter">Bpm</span><span className="text-amber-400 text-xs font-mono">{bpm}</span></div>
                                    <div className="px-3 py-0.5 border-r border-white/5 text-center"><span className="text-[6px] text-slate-500 block uppercase mb-0.5 tracking-tighter">Key</span><span className="text-blue-400 text-xs font-mono uppercase">{detectedKey}</span></div>
                                    <div className="px-3 py-0.5 text-center"><span className="text-[6px] text-slate-500 block uppercase mb-0.5 tracking-tighter">Timeline</span><span className="text-slate-200 text-xs font-mono">{formatTime(currentTime)}</span></div>
                                </div>
                            </div>
                        </header>

                        <main className="flex-1 overflow-y-auto custom-scrollbar p-4 space-y-4 pb-32">
                            {/* MASTER SCRUBBER */}
                            {masterTrack && (
                                <section className="bg-amber-500/5 border border-amber-500/20 rounded-2xl p-4 shadow-2xl overflow-hidden">
                                    <div className="flex items-center justify-between mb-3 px-1">
                                        <div className="flex items-center gap-2">
                                            <Waves size={14} className="text-amber-500" />
                                            <h2 className="text-[10px] font-black uppercase tracking-widest text-amber-500">Master Scrubber View</h2>
                                        </div>
                                        <span className="text-[9px] font-mono text-amber-500/50">
                                            {formatTime(currentTime)} <span className="mx-1">/</span> {formatTime(masterTrack.duration)}
                                        </span>
                                    </div>
                                    <div 
                                        className="h-20 bg-black/40 rounded-xl relative overflow-hidden cursor-pointer border border-white/5 group"
                                        onClick={(e) => handleSeek(e, masterTrack.duration)}
                                    >
                                        <div className="absolute inset-0 flex items-center justify-around px-2 opacity-40 group-hover:opacity-60 transition-opacity">
                                            {masterTrack.waveform.map((peak, i) => (
                                                <div key={i} className="w-[1px] bg-amber-500 rounded-full" style={{ height: `${peak * 90}%` }} />
                                            ))}
                                        </div>
                                        <div className="playhead-line" style={{ left: `${(currentTime / masterTrack.duration) * 100}%` }}>
                                            <div className="playhead-marker" />
                                        </div>
                                    </div>
                                </section>
                            )}

                            {/* TRACKS LIST */}
                            <div className="space-y-1.5">
                                {stems.map((track, idx) => (
                                    <div key={track.id} className="bg-white/[0.01] border border-white/5 rounded-xl p-3 flex flex-col lg:flex-row items-center gap-4 group hover:bg-white/[0.03] transition-all">
                                        <div className="w-full lg:w-64 shrink-0">
                                            <div className="flex items-center justify-between mb-1.5">
                                                <div className="flex items-center gap-2 min-w-0">
                                                    <div className={`w-7 h-7 bg-black/60 rounded border border-white/5 flex items-center justify-center ${track.color}`}>{track.type === 'full' ? <Waves size={14}/> : <Music size={14}/>}</div>
                                                    <span className="font-black text-[11px] truncate max-w-[100px]">{track.label}</span>
                                                </div>
                                                <div className="flex gap-1 shrink-0">
                                                    <button onClick={() => toggleSolo(track.id)} className={`w-6 h-6 rounded bg-black/40 text-[7px] font-black border border-white/5 transition-colors ${track.solo ? 'bg-amber-400 text-black border-amber-300 shadow-lg' : 'text-slate-500'}`}>S</button>
                                                    <button onClick={() => toggleMute(track.id)} className={`w-6 h-6 rounded bg-black/40 text-[7px] font-black border border-white/5 transition-colors ${track.muted ? 'bg-red-500 text-white border-red-400' : 'text-slate-500'}`}>M</button>
                                                </div>
                                            </div>
                                            <div className="flex items-center gap-2">
                                                <Volume2 size={12} className="text-slate-600" />
                                                <input type="range" min="-60" max="6" value={track.volume} onChange={(e) => handleVolumeChange(track.id, e.target.value)} className="flex-1" />
                                                <span className="text-[8px] font-mono text-amber-500 w-6 text-right">{track.volume}</span>
                                            </div>
                                        </div>

                                        <div className="flex-1 w-full bg-black/60 h-16 rounded border border-white/5 relative overflow-hidden cursor-crosshair group/track" onClick={(e) => handleSeek(e, track.duration)}>
                                            <div className="absolute inset-0 flex items-center justify-between px-4 opacity-25 group-hover/track:opacity-50 transition-opacity">
                                                {track.waveform && track.waveform.map((peak, pidx) => (
                                                    <div key={pidx} className={`w-[1px] rounded-full waveform-bar ${track.color.replace('text-', 'bg-')}`} style={{ height: `${Math.max(4, peak * 100)}%` }} />
                                                ))}
                                            </div>
                                            {track.duration > 0 && (
                                                <div className="playhead-line" style={{ left: `${(currentTime / track.duration) * 100}%` }} />
                                            )}
                                            {isAnalyzing && idx === 0 && (
                                                <div className="absolute inset-0 flex items-center justify-center bg-black/60 z-40">
                                                    <Loader2 size={16} className="animate-spin text-amber-500 mr-2" />
                                                    <span className="text-[8px] font-black uppercase tracking-widest text-amber-500">Processing Playlist...</span>
                                                </div>
                                            )}
                                        </div>
                                    </div>
                                ))}
                            </div>
                        </main>

                        {/* FOOTER CONTROL */}
                        <div className="absolute bottom-0 left-0 right-0 p-3 z-[100] pointer-events-none">
                            <div className="max-w-4xl mx-auto bg-black/95 backdrop-blur-2xl border border-white/10 rounded-2xl p-2 px-5 shadow-[0_30px_60px_rgba(0,0,0,1)] flex flex-col md:flex-row items-center gap-4 md:gap-6 pointer-events-auto border-t border-white/5">
                                
                                <div className="flex items-center gap-2 bg-white/[0.03] p-1 rounded-lg border border-white/5 shadow-inner">
                                    <button onClick={() => { Tone.Transport.stop(); setIsPlaying(false); setIsPaused(false); setCurrentTime(0); }} className="w-7 h-7 text-slate-500 hover:text-white rounded flex items-center justify-center transition-all active:scale-90"><Square size={14} fill="currentColor"/></button>
                                    <button onClick={() => { Tone.Transport.pause(); setIsPlaying(false); setIsPaused(true); }} className={`w-7 h-7 rounded flex items-center justify-center transition-all ${isPaused ? 'text-amber-400 bg-amber-500/10' : 'text-slate-500'}`}><Pause size={14} fill="currentColor"/></button>
                                    <button onClick={async () => { if (Tone.context.state !== 'running') await Tone.start(); Tone.Transport.start(); setIsPlaying(true); setIsPaused(false); }} className={`w-9 h-9 rounded flex items-center justify-center shadow-lg transition-all active:scale-95 ${isPlaying ? 'bg-amber-500 text-black shadow-amber-900/30' : 'bg-white/10 text-slate-400'}`}><Play size={18} fill="currentColor" className="ml-0.5"/></button>
                                </div>

                                <div className="flex-1 w-full flex items-center gap-3">
                                    <div className="shrink-0 flex items-center gap-1.5 min-w-[70px]">
                                        <span className="text-[7px] font-black uppercase text-slate-500 tracking-tighter">Speed</span>
                                        <span className="text-amber-500 font-mono text-[10px]">{Math.round(playbackRate*100)}%</span>
                                        <button onClick={() => setPlaybackRate(1)} className="text-slate-600 hover:text-amber-400" title="Reset Speed"><RotateCcw size={10}/></button>
                                    </div>
                                    <input type="range" min="0.5" max="1.5" step="0.01" value={playbackRate} onChange={(e) => setPlaybackRate(parseFloat(e.target.value))} className="flex-1" />
                                </div>

                                <div className="flex items-center gap-4 border-l border-white/10 pl-4">
                                    <div className="flex items-center gap-2 bg-black/60 px-2 py-1 rounded border border-white/5 relative group text-center">
                                        <button onClick={() => setTranspose(prev => prev - 1)} className="text-slate-500 hover:text-amber-400"><ChevronDown size={12}/></button>
                                        <span className="font-mono font-black text-amber-400 text-[10px] min-w-[12px]">{transpose > 0 ? `+${transpose}` : transpose}</span>
                                        <button onClick={() => setTranspose(prev => prev + 1)} className="text-slate-600 hover:text-amber-400"><ChevronUp size={12}/></button>
                                        {transpose !== 0 && <button onClick={() => setTranspose(0)} className="absolute -top-1.5 -right-1.5 bg-amber-500 text-black p-0.5 rounded-full shadow-lg scale-0 group-hover:scale-100 transition-all"><RotateCcw size={6}/></button>}
                                    </div>

                                    <div className="flex items-center gap-2">
                                        <div className="flex flex-col items-end">
                                            <button 
                                                onClick={() => setIsMetronomeDouble(!isMetronomeDouble)}
                                                className={`text-[7px] font-black px-1 rounded border mb-0.5 transition-all shadow-sm ${isMetronomeDouble ? 'bg-amber-500 text-black border-amber-400' : 'text-slate-600 border-white/5'}`}
                                                title="Tukar Kelajuan Metronom"
                                            >
                                                2X
                                            </button>
                                            <button 
                                                onClick={async () => { if (Tone.context.state !== 'running') await Tone.start(); setIsMetronomeOn(!isMetronomeOn); }} 
                                                className={`w-7 h-7 rounded border transition-all flex items-center justify-center ${isMetronomeOn ? 'bg-amber-500 text-black shadow-inner shadow-amber-900/20' : 'bg-white/5 border-white/5 text-slate-500'}`}
                                                title="Metronom Aktif"
                                            >
                                                <Activity size={14} className={isMetronomeOn ? 'animate-pulse' : ''} />
                                            </button>
                                        </div>
                                        <div className="flex flex-col gap-0.5">
                                            <span className="text-[5px] font-black text-slate-600 uppercase text-center tracking-tighter">Volume</span>
                                            <input type="range" min="-60" max="6" step="1" value={metronomeVolume} onChange={(e) => setMetronomeVolume(parseFloat(e.target.value))} className="w-12 h-1" />
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
