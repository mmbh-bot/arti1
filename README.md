
import React, { useState, useEffect, useCallback, useRef } from 'react';
import { analyzeScript, generateFrameImage, extractBackgroundDNA } from './services/geminiService';
import { StoryboardScene, StoryboardFrame, AspectRatio, Project, ProductionTier, BatchStatus, Character, BackgroundStyle, ProjectLogEntry } from './types';
import { Button } from './components/Button';
import { ChatWidget } from './components/ChatWidget';
import { LiveDirector } from './components/LiveDirector';
import JSZip from 'jszip';

declare global {
  interface AIStudio {
    hasSelectedApiKey: () => Promise<boolean>;
    openSelectKey: () => Promise<void>;
  }
  interface Window {
    aistudio?: AIStudio;
  }
}

const FlipbookPlayer: React.FC<{ frames: StoryboardFrame[]; playing?: boolean }> = ({ frames, playing = false }) => {
  const [currentIndex, setCurrentIndex] = useState(0);
  const timerRef = useRef<number | null>(null);
  const availableFrames = frames.filter(f => f.imageUrl);

  useEffect(() => {
    if (playing && availableFrames.length > 1) {
      timerRef.current = window.setInterval(() => {
        setCurrentIndex(prev => (prev + 1) % availableFrames.length);
      }, 1000 / 24);
    } else {
      if (timerRef.current) clearInterval(timerRef.current);
    }
    return () => { if (timerRef.current) clearInterval(timerRef.current); };
  }, [playing, availableFrames.length]);

  if (availableFrames.length === 0) {
    return (
      <div className="w-full h-full flex items-center justify-center bg-neutral-900 text-white/20 text-[10px] font-black uppercase tracking-widest">
        No Render Data
      </div>
    );
  }

  return (
    <div className="relative w-full h-full bg-white">
      <img 
        src={availableFrames[currentIndex].imageUrl} 
        alt="Flipbook frame" 
        className="w-full h-full object-contain"
      />
      <div className="absolute top-4 right-4 px-2 py-1 bg-black/80 text-white text-[8px] font-mono rounded backdrop-blur">
        {currentIndex + 1} / {availableFrames.length}
      </div>
    </div>
  );
};

export const App: React.FC = () => {
  const [projects, setProjects] = useState<Project[]>([]);
  const [activeProjectId, setActiveProjectId] = useState<string | null>(null);
  const [isSidebarOpen, setIsSidebarOpen] = useState(true);
  const [activeTab, setActiveTab] = useState<'script' | 'assets' | 'history'>('script');
  const [suggestedCharacters, setSuggestedCharacters] = useState<Character[]>([]);
  const [playingSceneId, setPlayingSceneId] = useState<string | null>(null);
  const [batchStatus, setBatchStatus] = useState<BatchStatus>({
    total: 0, current: 0, stage: 'IDLE', isPaused: false
  });

  const activeProject = projects.find(p => p.id === activeProjectId);

  // Persistence
  useEffect(() => {
    const savedProjects = localStorage.getItem('flipToon_projects_v4');
    if (savedProjects) {
      const parsed = JSON.parse(savedProjects);
      setProjects(parsed);
      if (parsed.length > 0) setActiveProjectId(parsed[0].id);
    } else {
      createNewProject();
    }
  }, []);

  useEffect(() => {
    if (projects.length > 0) localStorage.setItem('flipToon_projects_v4', JSON.stringify(projects));
  }, [projects]);

  const createNewProject = () => {
    const newProject: Project = {
      id: `proj-${Date.now()}`,
      name: `Project ${projects.length + 1}`,
      script: '',
      scenes: [],
      updatedAt: Date.now(),
      tier: ProductionTier.STANDARD,
      characters: [],
      history: [{ timestamp: Date.now(), message: 'Project created.' }]
    };
    setProjects(prev => [newProject, ...prev]);
    setActiveProjectId(newProject.id);
  };

  const addLogEntry = (projectId: string, message: string) => {
    setProjects(prev => prev.map(p => p.id === projectId ? {
      ...p,
      history: [{ timestamp: Date.now(), message }, ...p.history].slice(0, 50)
    } : p));
  };

  const updateActiveProject = (updates: Partial<Project>) => {
    setProjects(prev => prev.map(p => p.id === activeProjectId ? { ...p, ...updates, updatedAt: Date.now() } : p));
  };

  const handleCharUpload = (charId: string, e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      const base64 = reader.result as string;
      if (activeProject) {
        updateActiveProject({
          characters: activeProject.characters.map(c => c.id === charId ? { ...c, referenceImage: base64 } : c)
        });
        addLogEntry(activeProject.id, `Updated reference image for character.`);
      }
    };
    reader.readAsDataURL(file);
  };

  const handleBGUpload = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = async () => {
      const base64 = reader.result as string;
      setBatchStatus(prev => ({ ...prev, stage: 'EXTRACTING BACKGROUND STYLE...' }));
      try {
        const dna = await extractBackgroundDNA(base64);
        updateActiveProject({
          backgroundStyle: {
            id: `bg-${Date.now()}`,
            referenceImage: base64,
            description: dna.description || "Consistent cartoon",
            palette: dna.palette || "Vibrant flat colors",
            perspective: dna.perspective || "Flat 2D",
            lineStyle: dna.lineStyle || "Thick clean outlines"
          }
        });
        if (activeProject) addLogEntry(activeProject.id, `New background style reference uploaded.`);
      } catch (err) {
        console.error(err);
      } finally {
        setBatchStatus(prev => ({ ...prev, stage: 'IDLE' }));
      }
    };
    reader.readAsDataURL(file);
  };

  const handleStartProduction = async () => {
    if (!activeProject || !activeProject.script.trim()) return;
    setBatchStatus({ total: 0, current: 0, stage: 'ANALYZING SCRIPT...', isPaused: false });
    try {
      const { scenes, suggestedCharacters } = await analyzeScript(activeProject.script, activeProject.characters, activeProject.backgroundStyle, true);
      updateActiveProject({ scenes });
      setSuggestedCharacters(suggestedCharacters);
      addLogEntry(activeProject.id, `Script analyzed. ${scenes.length} shots generated.`);
      
      const totalFrames = scenes.reduce((acc, s) => acc + s.frames.length, 0);
      setBatchStatus({ total: totalFrames, current: 0, stage: 'PRODUCTION INITIALIZED', isPaused: false });
      setTimeout(() => startRenderingLoop(scenes), 100);
    } catch (err: any) {
      setBatchStatus(prev => ({ ...prev, stage: 'ANALYSIS FAILED', errorType: err.type }));
    }
  };

  const startRenderingLoop = async (currentScenes?: StoryboardScene[]) => {
    const scenesToProcess = currentScenes || activeProject?.scenes;
    if (!scenesToProcess || batchStatus.isPaused) return;
    let currentRendered = 0;
    const projectTier = activeProject?.tier || ProductionTier.STANDARD;

    for (const scene of scenesToProcess) {
      for (const frame of scene.frames) {
        if (frame.imageUrl) { currentRendered++; continue; }
        setBatchStatus(prev => ({ 
          ...prev, 
          stage: `SCENE ${scene.sceneNumber} | FRAME ${scene.frames.indexOf(frame) + 1}`,
          current: currentRendered 
        }));
        try {
          frame.isGenerating = true;
          updateActiveProject({ scenes: [...scenesToProcess] });
          const url = await generateFrameImage(
            frame.visualPrompt,
            AspectRatio.RATIO_16_9,
            projectTier,
            activeProject?.characters || [],
            activeProject?.backgroundStyle,
            scene.backgroundContext
          );
          frame.imageUrl = url;
          frame.isGenerating = false;
          currentRendered++;
          updateActiveProject({ scenes: [...scenesToProcess] });
        } catch (err: any) {
          frame.isGenerating = false;
          frame.error = err.message;
          updateActiveProject({ scenes: [...scenesToProcess] });
          if (err.type === 'QUOTA') {
            setBatchStatus(prev => ({ ...prev, isPaused: true, errorType: 'QUOTA', stage: 'QUOTA LIMIT REACHED' }));
            return;
          }
        }
      }
    }
    setBatchStatus(prev => ({ ...prev, stage: 'PRODUCTION COMPLETE', current: prev.total }));
    if (activeProject) addLogEntry(activeProject.id, `Batch production complete.`);
  };

  return (
    <div className="min-h-screen bg-[#050505] text-white font-outfit flex overflow-hidden">
      {/* Sidebar */}
      <aside className={`transition-all duration-300 border-r border-white/5 bg-black z-30 flex flex-col ${isSidebarOpen ? 'w-80' : 'w-0 overflow-hidden'}`}>
        <div className="p-8 flex justify-between items-center whitespace-nowrap">
          <span className="text-xs font-black tracking-widest uppercase opacity-40">Projects</span>
          <button onClick={createNewProject} className="w-8 h-8 rounded-full bg-white text-black flex items-center justify-center hover:scale-110 transition-all">+</button>
        </div>
        <div className="flex-1 overflow-y-auto px-4 space-y-2">
          {projects.map(p => (
            <div 
              key={p.id} 
              onClick={() => setActiveProjectId(p.id)} 
              className={`group flex items-center justify-between p-4 rounded-2xl cursor-pointer transition-all border ${activeProjectId === p.id ? 'bg-white/10 border-white/10' : 'bg-transparent border-transparent hover:bg-white/5'}`}
            >
              <div className="flex-1 min-w-0">
                <p className="text-sm font-bold truncate">{p.name || 'Untitled'}</p>
                <p className="text-[10px] text-white/20 uppercase mt-1">{new Date(p.updatedAt).toLocaleDateString()}</p>
              </div>
            </div>
          ))}
        </div>
      </aside>

      {/* Main Content */}
      <div className="flex-1 flex flex-col overflow-hidden relative">
        {batchStatus.stage !== 'IDLE' && (
          <div className={`h-12 flex items-center justify-between px-12 transition-all ${batchStatus.errorType === 'QUOTA' ? 'bg-red-600' : 'bg-blue-600'}`}>
            <span className="text-[10px] font-black tracking-widest uppercase animate-pulse">{batchStatus.stage}</span>
            <div className="flex-1 mx-12 h-1 bg-black/20 rounded-full overflow-hidden">
              <div className="h-full bg-white transition-all duration-500" style={{ width: `${(batchStatus.current / batchStatus.total) * 100 || 0}%` }} />
            </div>
            <span className="text-[10px] font-mono opacity-60">{batchStatus.current} / {batchStatus.total}</span>
          </div>
        )}

        <nav className="h-20 border-b border-white/5 px-12 flex items-center justify-between bg-black/40 backdrop-blur-xl">
          <div className="flex items-center gap-6">
            <button onClick={() => setIsSidebarOpen(!isSidebarOpen)} className="p-2 text-white/40 hover:text-white transition-colors">
              <svg className="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M4 6h16M4 12h16M4 18h16" /></svg>
            </button>
            <input 
              value={activeProject?.name || ''} 
              onChange={(e) => updateActiveProject({ name: e.target.value })} 
              className="bg-transparent text-xl font-black tracking-tighter focus:outline-none placeholder:opacity-20" 
              placeholder="Name your production..." 
            />
          </div>
          <div className="flex items-center gap-8">
            <div className="flex gap-6">
              <button onClick={() => setActiveTab('script')} className={`text-[10px] font-black uppercase tracking-[0.2em] transition-all ${activeTab === 'script' ? 'text-white' : 'text-white/30 hover:text-white/60'}`}>Script</button>
              <button onClick={() => setActiveTab('assets')} className={`text-[10px] font-black uppercase tracking-[0.2em] transition-all ${activeTab === 'assets' ? 'text-white' : 'text-white/30 hover:text-white/60'}`}>Assets</button>
              <button onClick={() => setActiveTab('history')} className={`text-[10px] font-black uppercase tracking-[0.2em] transition-all ${activeTab === 'history' ? 'text-white' : 'text-white/30 hover:text-white/60'}`}>Log</button>
            </div>
            <div className="flex bg-white/5 p-1 rounded-full border border-white/5">
              {[ProductionTier.LITE, ProductionTier.STANDARD, ProductionTier.PRO].map(t => (
                <button key={t} onClick={() => updateActiveProject({ tier: t })} className={`px-4 py-1.5 rounded-full text-[10px] font-black transition-all ${activeProject?.tier === t ? 'bg-white text-black' : 'text-white/40 hover:text-white'}`}>{t}</button>
              ))}
            </div>
          </div>
        </nav>

        <main className="flex-1 overflow-y-auto p-12 space-y-12">
          {!activeProject ? (
            <div className="h-full flex flex-col items-center justify-center opacity-20 text-center"><h1 className="text-4xl font-black">SELECT A PROJECT</h1></div>
          ) : activeTab === 'script' ? (
            <div className="grid grid-cols-12 gap-12">
              <div className="col-span-4 space-y-6">
                <div className="glass p-10 rounded-[3rem] space-y-6">
                  <div className="flex justify-between items-center">
                    <h3 className="text-xs font-black tracking-widest text-white/30 uppercase">Master Script</h3>
                    <div className="text-[10px] text-blue-400 font-bold uppercase">FREE TIER GEN: ON</div>
                  </div>
                  <textarea 
                    value={activeProject.script} 
                    onChange={(e) => updateActiveProject({ script: e.target.value })} 
                    className="w-full h-[550px] bg-white/5 border border-white/10 rounded-[2rem] p-8 text-sm focus:outline-none focus:ring-2 ring-white/5 resize-none font-mono leading-relaxed" 
                    placeholder="Describe your story. Character names should be consistent." 
                  />
                  <Button onClick={handleStartProduction} className="w-full py-6 text-lg rounded-2xl shadow-xl">Launch Production</Button>
                </div>
              </div>

              <div className="col-span-8 space-y-12">
                {suggestedCharacters.length > 0 && (
                  <div className="p-8 bg-blue-500/10 border border-blue-500/20 rounded-[3rem] space-y-6">
                    <h4 className="text-xs font-black uppercase tracking-widest text-blue-400">Supporting Characters Inferred</h4>
                    <div className="grid grid-cols-2 gap-4">
                      {suggestedCharacters.map(c => (
                        <div key={c.id} className="p-4 bg-black/40 rounded-2xl border border-white/5 flex justify-between items-center">
                          <div>
                            <p className="text-sm font-bold">{c.name}</p>
                            <p className="text-[10px] opacity-40 italic truncate max-w-[150px]">{c.description}</p>
                          </div>
                          <Button variant="ghost" className="text-[10px] py-1" onClick={() => {
                            updateActiveProject({ characters: [...activeProject.characters, c] });
                            setSuggestedCharacters(prev => prev.filter(sc => sc.id !== c.id));
                            addLogEntry(activeProject.id, `Added suggested character: ${c.name}`);
                          }}>Approve</Button>
                        </div>
                      ))}
                    </div>
                  </div>
                )}

                {activeProject.scenes.map(s => (
                  <div key={s.id} className="glass rounded-[4rem] overflow-hidden border border-white/5">
                    <div className="flex flex-col xl:flex-row">
                      <div className="xl:w-2/3 aspect-video bg-black relative group">
                        <FlipbookPlayer frames={s.frames} playing={playingSceneId === s.id} />
                        <div className="absolute inset-0 bg-black/60 opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center">
                          <button onClick={() => setPlayingSceneId(playingSceneId === s.id ? null : s.id)} className="w-20 h-20 rounded-full bg-white text-black flex items-center justify-center shadow-2xl hover:scale-110 active:scale-95 transition-all">
                            {playingSceneId === s.id ? <svg className="w-8 h-8" fill="currentColor" viewBox="0 0 24 24"><path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/></svg> : <svg className="w-8 h-8 ml-1" fill="currentColor" viewBox="0 0 24 24"><path d="M8 5v14l11-7z"/></svg>}
                          </button>
                        </div>
                      </div>
                      <div className="xl:w-1/3 p-12 flex flex-col justify-between bg-white/[0.01]">
                        <div>
                          <span className="px-3 py-1 bg-white/10 text-white/40 text-[8px] font-black rounded-full uppercase tracking-widest">Shot {s.sceneNumber}</span>
                          <h4 className="mt-4 text-xs font-black uppercase text-white/20 tracking-widest">{s.location}</h4>
                          <p className="text-lg font-light italic text-neutral-400 mt-4 leading-relaxed">"{s.description}"</p>
                          <div className="mt-8 p-4 bg-blue-500/5 border border-blue-500/10 rounded-2xl">
                            <p className="text-[9px] text-blue-400 font-bold uppercase tracking-widest mb-1">Dynamic BG</p>
                            <p className="text-[10px] text-blue-300/60 italic leading-tight">{s.backgroundContext}</p>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          ) : activeTab === 'assets' ? (
            <div className="max-w-5xl mx-auto space-y-12">
              <section className="glass p-12 rounded-[4rem] space-y-12">
                <div className="flex justify-between items-center">
                  <div>
                    <h3 className="text-2xl font-black tracking-tighter">Character Library</h3>
                    <p className="text-neutral-500 text-sm mt-1">Updates propagate to future renders.</p>
                  </div>
                  <Button variant="secondary" onClick={() => {
                    const newChar: Character = { id: `char-${Date.now()}`, name: 'New Character', description: 'Cartoon persona' };
                    updateActiveProject({ characters: [...activeProject.characters, newChar] });
                    addLogEntry(activeProject.id, `Manual character created.`);
                  }}>Add Character</Button>
                </div>
                <div className="grid grid-cols-2 gap-8">
                  {activeProject.characters.map(c => (
                    <div key={c.id} className="p-8 bg-white/5 rounded-[3rem] border border-white/5 space-y-6 group relative">
                      <div className="flex gap-6">
                        <div className="w-32 h-32 bg-black rounded-3xl border border-white/10 flex items-center justify-center overflow-hidden relative shadow-inner">
                          {c.referenceImage ? (
                            <img src={c.referenceImage} alt={c.name} className="w-full h-full object-contain" />
                          ) : (
                            <div className="text-[8px] font-black opacity-20 uppercase">Ref Image</div>
                          )}
                          <input type="file" onChange={(e) => handleCharUpload(c.id, e)} className="absolute inset-0 opacity-0 cursor-pointer" />
                        </div>
                        <div className="flex-1 space-y-4">
                          <input 
                            value={c.name} 
                            onChange={(e) => updateActiveProject({ characters: activeProject.characters.map(ch => ch.id === c.id ? { ...ch, name: e.target.value } : ch) })} 
                            className="bg-transparent text-xl font-black tracking-tight border-none focus:outline-none w-full" 
                            placeholder="Character Name" 
                          />
                          <textarea 
                            value={c.description} 
                            onChange={(e) => updateActiveProject({ characters: activeProject.characters.map(ch => ch.id === c.id ? { ...ch, description: e.target.value } : ch) })} 
                            className="w-full bg-white/5 border border-white/10 rounded-2xl p-4 text-xs focus:outline-none resize-none leading-relaxed" 
                            rows={3} 
                            placeholder="Visual description (hair, clothing)..." 
                          />
                        </div>
                      </div>
                      <button 
                        onClick={() => updateActiveProject({ characters: activeProject.characters.filter(ch => ch.id !== c.id) })} 
                        className="absolute top-4 right-8 text-[10px] font-bold text-red-500/40 hover:text-red-500 uppercase tracking-widest transition-colors"
                      >
                        Remove
                      </button>
                    </div>
                  ))}
                </div>
              </section>

              <section className="glass p-12 rounded-[4rem] space-y-10">
                <h3 className="text-2xl font-black tracking-tighter">Global Background Style</h3>
                <div className="flex gap-12">
                  <div className="w-64 h-64 bg-black rounded-[3rem] border border-white/10 flex items-center justify-center overflow-hidden relative group shadow-2xl">
                    {activeProject.backgroundStyle?.referenceImage ? (
                      <img src={activeProject.backgroundStyle.referenceImage} alt="BG DNA" className="w-full h-full object-cover" />
                    ) : (
                      <div className="text-center p-6 space-y-2">
                        <p className="text-[8px] opacity-20 uppercase font-black tracking-widest">Upload Style DNA</p>
                      </div>
                    )}
                    <input type="file" onChange={handleBGUpload} className="absolute inset-0 opacity-0 cursor-pointer" />
                  </div>
                  <div className="flex-1 space-y-6 pt-4">
                    {activeProject.backgroundStyle ? (
                      <div className="space-y-4">
                        <p className="text-xs uppercase font-black tracking-[0.2em] text-blue-400">Dynamic Background Engine: ACTIVE</p>
                        <div className="p-6 bg-white/5 rounded-2xl border border-white/5 text-sm italic text-white/60 leading-relaxed">
                          {activeProject.backgroundStyle.description}
                        </div>
                        <Button variant="ghost" onClick={() => updateActiveProject({ backgroundStyle: undefined })} className="text-[10px] opacity-40 hover:opacity-100">Reset DNA</Button>
                      </div>
                    ) : (
                      <p className="text-sm opacity-30 italic leading-relaxed">Upload a reference background. The system will extract the color palette, line style, and geometry to generate all future scenes automatically.</p>
                    )}
                  </div>
                </div>
              </section>
            </div>
          ) : (
            <div className="max-w-3xl mx-auto glass p-12 rounded-[4rem] space-y-8">
              <h3 className="text-2xl font-black tracking-tighter">Production Log</h3>
              <div className="space-y-4">
                {activeProject.history.map((log, idx) => (
                  <div key={idx} className="flex gap-6 items-start text-sm border-b border-white/5 pb-4 last:border-0">
                    <span className="text-[10px] font-mono opacity-20 pt-1 whitespace-nowrap">{new Date(log.timestamp).toLocaleTimeString()}</span>
                    <p className="opacity-60">{log.message}</p>
                  </div>
                ))}
              </div>
            </div>
          )}
        </main>
      </div>

      <ChatWidget scriptContext={activeProject?.script || ''} />
      <LiveDirector scriptContext={activeProject?.script || ''} />
    </div>
  );
};

export default App;
