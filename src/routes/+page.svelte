<script lang="ts">
	import { onMount } from 'svelte';
	// ── 타입 ──────────────────────────────────────────────────────────────
	type TextObject = {
		kind: 'text';
		id: string;
		x: number;
		y: number;
		content: string;
		color: string;
		fontSize: number;
		bold: boolean;
		italic: boolean;
		underline: boolean;
		alpha: number;
		locked: boolean;
	};

	type LineObject = {
		kind: 'line';
		id: string;
		x1: number;
		y1: number;
		x2: number;
		y2: number;
		color: string;
		strokeWidth: number;
		alpha: number;
		locked: boolean;
	};

	type ScribObject = TextObject | LineObject;

	// ── 상태 ──────────────────────────────────────────────────────────────
	let objects = $state<ScribObject[]>([]);
	let history = $state<string[]>([]); // JSON 스냅샷 스택
	let future = $state<string[]>([]);
	let idCounter = $state(0);

	// 캔버스 변환 — 브라우저에서만 초기화 (아래 $effect 참고)
	let offsetX = $state(0);
	let offsetY = $state(0);
	let zoom = $state(1);
	const GRID = 24; // 그리드 한 칸 픽셀

	// 선택
	let selectedIds = $state<Set<string>>(new Set());

	// 텍스트 입력 커서 (좌표 없이 입력 시 여기에 배치)
	let cursorX = $state(0);
	let cursorY = $state(0);

	// 입력
	let inputValue = $state('');
	let cmdHistory = $state<string[]>([]);
	let cmdHistIdx = $state(-1);
	let inputEl: HTMLInputElement;

	// 패닝
	let isPanning = $state(false);
	let panStart = { x: 0, y: 0 };

	// 드래그 이동
	let isDragging = $state(false);
	let dragStartGrid = { x: 0, y: 0 };
	type OrigPos = { x: number; y: number } | { x1: number; y1: number; x2: number; y2: number };
	let dragOrigPos = $state<Map<string, OrigPos>>(new Map());

	// ── 36진수 ID ─────────────────────────────────────────────────────────
	function nextId() {
		return (idCounter++).toString(36);
	}

	// ── 히스토리 ──────────────────────────────────────────────────────────
	function snapshot() {
		history = [...history, JSON.stringify(objects)];
		future = [];
	}
	function undo() {
		if (!history.length) return;
		future = [...future, JSON.stringify(objects)];
		objects = JSON.parse(history[history.length - 1]);
		history = history.slice(0, -1);
		selectedIds = new Set();
	}
	function redo() {
		if (!future.length) return;
		history = [...history, JSON.stringify(objects)];
		objects = JSON.parse(future[future.length - 1]);
		future = future.slice(0, -1);
		selectedIds = new Set();
	}

	// ── 좌표 변환 ─────────────────────────────────────────────────────────
	// 픽셀 → 그리드
	function toGrid(px: number, py: number) {
		return {
			x: Math.round((px - offsetX) / (GRID * zoom)),
			y: Math.round((py - offsetY) / (GRID * zoom))
		};
	}
	// 그리드 → 픽셀 (SVG 내부 좌표)
	function gx(x: number) {
		return x * GRID;
	}
	function gy(y: number) {
		return y * GRID;
	}

	// ── 오브젝트 셀렉터 ───────────────────────────────────────────────────
	// "id 또는 x1,y1-x2,y2 범위"를 파싱하여 대상 ID 목록 반환
	function resolveTarget(token: string): string[] {
		const range = token.match(/^(-?\d+),(-?\d+)-(-?\d+),(-?\d+)$/);
		if (range) {
			const [, x1, y1, x2, y2] = range.map(Number);
			const minX = Math.min(x1, x2),
				maxX = Math.max(x1, x2);
			const minY = Math.min(y1, y2),
				maxY = Math.max(y1, y2);
			return objects
				.filter((o) => {
					if (o.kind === 'text') return o.x >= minX && o.x <= maxX && o.y >= minY && o.y <= maxY;
					return (
						(o.x1 >= minX && o.x1 <= maxX && o.y1 >= minY && o.y1 <= maxY) ||
						(o.x2 >= minX && o.x2 <= maxX && o.y2 >= minY && o.y2 <= maxY)
					);
				})
				.map((o) => o.id);
		}
		// 단일 ID
		return objects.find((o) => o.id === token) ? [token] : [];
	}

	// ── 커맨드 파서 ───────────────────────────────────────────────────────
	function execute(raw: string) {
		const input = raw.trim();
		if (!input) return;

		// 히스토리 저장
		cmdHistory = [input, ...cmdHistory.filter((h) => h !== input)].slice(0, 50);
		cmdHistIdx = -1;

		// ─ 검색: /검색어
		if (input.startsWith('/')) {
			const q = input.slice(1);
			selectedIds = new Set(
				objects.filter((o) => o.kind === 'text' && o.content.includes(q)).map((o) => o.id)
			);
			return;
		}

		// ─ 커맨드: :로 시작
		if (input.startsWith(':')) {
			const rest = input.slice(1).trim();
			const parts = rest.match(/\S+/g) ?? [];
			const cmd = parts[0] ?? '';
			const args = parts.slice(1);
			runCmd(cmd, args, rest);
			return;
		}

		// ─ 텍스트 생성: "내용 @x,y" 또는 좌표 없이 "내용"
		const m = input.match(/^(.*?)\s+@(-?\d+),(-?\d+)$/);
		const coordOnly = input.match(/^@(-?\d+),(-?\d+)$/); // "@x,y"만 입력 → 커서 이동
		if (coordOnly) {
			cursorX = +coordOnly[1];
			cursorY = +coordOnly[2];
		} else {
			const fontSize = 16;
			const lineAdvance = Math.max(1, Math.ceil((fontSize * 1.4) / GRID));
			snapshot();
			const id = nextId();
			const x = m ? +m[2] : cursorX;
			const y = m ? +m[3] : cursorY;
			objects = [
				...objects,
				{
					kind: 'text',
					id,
					x,
					y,
					content: m ? m[1] : input,
					color: '#111111',
					fontSize,
					bold: false,
					italic: false,
					underline: false,
					alpha: 100,
					locked: false
				}
			];
			selectedIds = new Set(); // 추가 후 선택 강조 없음
			cursorX = x;
			cursorY = y + lineAdvance;
		}
	}

	function runCmd(cmd: string, args: string[], rest: string) {
		switch (cmd) {
			// ── 실행 취소 / 다시 실행 ───────────────────────────────
			case 'u':
				if (!args.length) {
					undo();
					return;
				}
				// 인수 있으면 underline (아래 스타일 처리에서 계속)
				styleToggle(args[0], 'underline');
				return;
			case 'r':
				if (!args.length) {
					redo();
					return;
				}
				// 인수 있으면 replace
				replaceContent(args[0], args.slice(1).join(' '));
				return;
			case 're':
				redo();
				return;

			// ── 생성 ────────────────────────────────────────────────
			case 'l': {
				const ms = [...rest.matchAll(/@(-?\d+),(-?\d+)/g)];
				if (ms.length < 2) return;
				snapshot();
				const id = nextId();
				objects = [
					...objects,
					{
						kind: 'line',
						id,
						x1: +ms[0][1],
						y1: +ms[0][2],
						x2: +ms[1][1],
						y2: +ms[1][2],
						color: '#111111',
						strokeWidth: 1,
						alpha: 100,
						locked: false
					}
				];
				selectedIds = new Set([id]);
				return;
			}

			// ── 편집 ────────────────────────────────────────────────
			case 'd': {
				if (!args[0]) return;
				const ids = resolveTarget(args[0]);
				if (!ids.length) return;
				snapshot();
				objects = objects.filter((o) => !ids.includes(o.id));
				selectedIds = new Set();
				return;
			}
			case 'm': {
				if (!args[0]) return;
				const ids = resolveTarget(args[0]);
				const pm = rest.match(/@(-?\d+),(-?\d+)/);
				if (!ids.length || !pm) return;
				snapshot();
				const tx = +pm[1],
					ty = +pm[2];
				// 단일: 절대 이동. 범위: bounding box 좌상단 기준
				if (ids.length === 1) {
					objects = objects.map((o) =>
						o.id === ids[0]
							? o.kind === 'text'
								? { ...o, x: tx, y: ty }
								: { ...o, x1: tx, y1: ty, x2: o.x2 + (tx - o.x1), y2: o.y2 + (ty - o.y1) }
							: o
					);
				} else {
					const targets = objects.filter((o) => ids.includes(o.id));
					const minX = Math.min(
						...targets.map((o) => (o.kind === 'text' ? o.x : Math.min(o.x1, o.x2)))
					);
					const minY = Math.min(
						...targets.map((o) => (o.kind === 'text' ? o.y : Math.min(o.y1, o.y2)))
					);
					const dx = tx - minX,
						dy = ty - minY;
					objects = objects.map((o) => {
						if (!ids.includes(o.id)) return o;
						if (o.kind === 'text') return { ...o, x: o.x + dx, y: o.y + dy };
						return { ...o, x1: o.x1 + dx, y1: o.y1 + dy, x2: o.x2 + dx, y2: o.y2 + dy };
					});
				}
				return;
			}
			case 'cp': {
				if (!args[0]) return;
				const ids = resolveTarget(args[0]);
				const pm = rest.match(/@(-?\d+),(-?\d+)/);
				if (!ids.length || !pm) return;
				snapshot();
				const tx = +pm[1],
					ty = +pm[2];
				const targets = objects.filter((o) => ids.includes(o.id));
				const minX = Math.min(
					...targets.map((o) => (o.kind === 'text' ? o.x : Math.min(o.x1, o.x2)))
				);
				const minY = Math.min(
					...targets.map((o) => (o.kind === 'text' ? o.y : Math.min(o.y1, o.y2)))
				);
				const dx = tx - minX,
					dy = ty - minY;
				const newObjs = targets.map((o) => {
					const id = nextId();
					if (o.kind === 'text') return { ...o, id, x: o.x + dx, y: o.y + dy };
					return { ...o, id, x1: o.x1 + dx, y1: o.y1 + dy, x2: o.x2 + dx, y2: o.y2 + dy };
				});
				objects = [...objects, ...newObjs];
				selectedIds = new Set(newObjs.map((o) => o.id));
				return;
			}

			// ── 스타일 ──────────────────────────────────────────────
			case 'c': {
				// :c <id|범위> <color>  또는 undo (인수 없을 때 - 위에서 처리)
				if (!args[0] || !args[1]) return;
				const ids = resolveTarget(args[0]);
				if (!ids.length) return;
				snapshot();
				objects = objects.map((o) => (ids.includes(o.id) ? { ...o, color: args[1] } : o));
				return;
			}
			case 's': {
				if (!args[0] || !args[1]) return;
				const ids = resolveTarget(args[0]);
				const sz = +args[1];
				if (!ids.length || isNaN(sz)) return;
				snapshot();
				objects = objects.map((o) =>
					ids.includes(o.id) && o.kind === 'text' ? { ...o, fontSize: sz } : o
				);
				return;
			}
			case 'b':
				styleToggle(args[0], 'bold');
				return;
			case 'i':
				styleToggle(args[0], 'italic');
				return;
			case 'w': {
				// :w 단독 → 저장 / :w <id> <굵기> → strokeWidth
				if (!args[0] || isNaN(+args[1])) {
					saveFile(args[0]);
				} else {
					const ids = resolveTarget(args[0]);
					const w = +args[1];
					if (!ids.length || isNaN(w)) return;
					snapshot();
					objects = objects.map((o) =>
						ids.includes(o.id) && o.kind === 'line' ? { ...o, strokeWidth: w } : o
					);
				}
				return;
			}
			case 'a': {
				if (!args[0] || !args[1]) return;
				const ids = resolveTarget(args[0]);
				const v = Math.max(0, Math.min(100, +args[1]));
				if (!ids.length || isNaN(v)) return;
				snapshot();
				objects = objects.map((o) => (ids.includes(o.id) ? { ...o, alpha: v } : o));
				return;
			}
			case 'lock': {
				const ids = resolveTarget(args[0] ?? '');
				if (!ids.length) return;
				objects = objects.map((o) => (ids.includes(o.id) ? { ...o, locked: true } : o));
				return;
			}
			case 'unlock': {
				const ids = resolveTarget(args[0] ?? '');
				if (!ids.length) return;
				objects = objects.map((o) => (ids.includes(o.id) ? { ...o, locked: false } : o));
				return;
			}

			// ── 레이어 ──────────────────────────────────────────────
			case 'top': {
				const o = objects.find((o) => o.id === args[0]);
				if (!o) return;
				objects = [...objects.filter((o) => o.id !== args[0]), o];
				return;
			}
			case 'bot': {
				const o = objects.find((o) => o.id === args[0]);
				if (!o) return;
				objects = [o, ...objects.filter((o) => o.id !== args[0])];
				return;
			}
			case 'up': {
				const idx = objects.findIndex((o) => o.id === args[0]);
				if (idx < 0 || idx === objects.length - 1) return;
				const arr = [...objects];
				[arr[idx], arr[idx + 1]] = [arr[idx + 1], arr[idx]];
				objects = arr;
				return;
			}
			case 'dn': {
				const idx = objects.findIndex((o) => o.id === args[0]);
				if (idx <= 0) return;
				const arr = [...objects];
				[arr[idx], arr[idx - 1]] = [arr[idx - 1], arr[idx]];
				objects = arr;
				return;
			}

			// ── 파일 ────────────────────────────────────────────────
			case 'wq':
				saveFile(args[0]);
				/* TODO: 종료 */ return;
			case 'q':
				/* TODO: 종료 확인 */ return;
			case 'q!':
				/* TODO: 강제 종료 */ return;
			case 'e':
				loadFile(args[0]);
				return;
			case 'x':
				exportImage(args[0] ?? 'png');
				return;

			// ── 캔버스 ──────────────────────────────────────────────
			case 'z': {
				// :z <퍼센트> → 절대 줌
				const pct = +args[0];
				if (!isNaN(pct) && pct > 0) zoom = pct / 100;
				return;
			}
			default: {
				// :z+ / :z++ / :z- / :z--- 등 → 상대 줌 (한 단계 ×1.2)
				const zm = cmd.match(/^z([+\-]+)$/);
				if (zm) {
					const delta = [...zm[1]].reduce((s, c) => s + (c === '+' ? 1 : -1), 0);
					zoom = Math.max(0.05, Math.min(20, zoom * Math.pow(1.2, delta)));
				}
				return;
			}
			case 'g': {
				showGrid = args[0] === 'off' ? false : args[0] === 'on' ? true : !showGrid;
				return;
			}
			case '0':
				offsetX = viewW / 2;
				offsetY = viewH / 2;
				zoom = 1;
				return;
			case 'f':
				fitAll();
				return;
			case 'pv':
				showIds = !showIds;
				return;
		}
	}

	function replaceContent(id: string, content: string) {
		if (!id || !content) return;
		snapshot();
		objects = objects.map((o) => (o.id === id && o.kind === 'text' ? { ...o, content } : o));
	}

	function styleToggle(target: string, prop: 'bold' | 'italic' | 'underline') {
		if (!target) return;
		const ids = resolveTarget(target);
		if (!ids.length) return;
		snapshot();
		objects = objects.map((o) =>
			ids.includes(o.id) && o.kind === 'text' ? { ...o, [prop]: !o[prop] } : o
		);
	}

	// ── 저장 / 불러오기 ───────────────────────────────────────────────────
	function saveFile(name?: string) {
		const data = JSON.stringify({ objects, idCounter }, null, 2);
		const blob = new Blob([data], { type: 'application/json' });
		const a = document.createElement('a');
		a.href = URL.createObjectURL(blob);
		a.download = (name ?? 'scribbler') + '.json';
		a.click();
	}
	function loadFile(name?: string) {
		const input = document.createElement('input');
		input.type = 'file';
		input.accept = '.json';
		input.onchange = async () => {
			const file = input.files?.[0];
			if (!file) return;
			const text = await file.text();
			const data = JSON.parse(text);
			snapshot();
			objects = data.objects ?? [];
			idCounter = data.idCounter ?? 0;
		};
		input.click();
	}

	// ── 이미지 내보내기 ───────────────────────────────────────────────────
	function exportImage(fmt: string) {
		const svgEl = document.querySelector('svg.canvas') as SVGSVGElement;
		if (!svgEl) return;
		const xml = new XMLSerializer().serializeToString(svgEl);
		const blob = new Blob([xml], { type: 'image/svg+xml' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `scribbler.${fmt === 'svg' ? 'svg' : 'png'}`;
		a.click();
	}

	// ── Fit 뷰 ────────────────────────────────────────────────────────────
	function fitAll() {
		if (!objects.length) return;
		const xs: number[] = [],
			ys: number[] = [];
		for (const o of objects) {
			if (o.kind === 'text') {
				const lines = o.content.split('\\\\');
				const w =
					Math.max(...lines.map((l) => measureLine(l, o.fontSize, o.bold, o.italic))) / GRID;
				const h = ((lines.length - 1) * o.fontSize * 1.4 + o.fontSize * 1.2) / GRID;
				xs.push(o.x, o.x + w);
				ys.push(o.y - o.fontSize / GRID, o.y + h - o.fontSize / GRID);
			} else {
				xs.push(o.x1, o.x2);
				ys.push(o.y1, o.y2);
			}
		}
		const pad = 2;
		const minX = Math.min(...xs) - pad,
			maxX = Math.max(...xs) + pad;
		const minY = Math.min(...ys) - pad,
			maxY = Math.max(...ys) + pad;
		const W = window.innerWidth,
			H = window.innerHeight - 60;
		const scaleX = W / ((maxX - minX) * GRID);
		const scaleY = H / ((maxY - minY) * GRID);
		zoom = Math.min(scaleX, scaleY);
		offsetX = (W - (maxX - minX) * GRID * zoom) / 2 - minX * GRID * zoom;
		offsetY = (H - (maxY - minY) * GRID * zoom) / 2 - minY * GRID * zoom;
	}

	// ── UI 상태 ───────────────────────────────────────────────────────────
	let showGrid = $state(true);
	let showIds = $state(true);

	// ── 키보드 ────────────────────────────────────────────────────────────
	function onKeydown(e: KeyboardEvent) {
		if (e.key === 'Enter') {
			execute(inputValue);
			inputValue = '';
		} else if (e.key === 'ArrowUp') {
			e.preventDefault();
			const next = Math.min(cmdHistIdx + 1, cmdHistory.length - 1);
			cmdHistIdx = next;
			if (cmdHistory[next]) inputValue = cmdHistory[next];
		} else if (e.key === 'ArrowDown') {
			e.preventDefault();
			const next = Math.max(cmdHistIdx - 1, -1);
			cmdHistIdx = next;
			inputValue = next < 0 ? '' : cmdHistory[next];
		} else if (e.ctrlKey && e.key === 'z') {
			e.preventDefault();
			undo();
		} else if (e.ctrlKey && e.key === 'y') {
			e.preventDefault();
			redo();
		} else if (e.ctrlKey && e.key === 's') {
			e.preventDefault();
			saveFile();
		}
	}

	// ── 마우스: 캔버스 클릭 / 패닝 / 드래그 ─────────────────────────────
	function onCanvasMousedown(e: MouseEvent) {
		if (e.button === 1 || (e.button === 0 && e.altKey)) {
			// 패닝
			isPanning = true;
			panStart = { x: e.clientX - offsetX, y: e.clientY - offsetY };
			e.preventDefault();
			return;
		}
		if (e.button === 0) {
			const { x, y } = toGrid(e.clientX, e.clientY);
			// 빈 공간 클릭 → 커서 이동 + @x,y 입력창에
			selectedIds = new Set();
			cursorX = x;
			cursorY = y;
			inputValue = `@${x},${y}`;
			inputEl?.focus();
		}
	}
	function onCanvasMousemove(e: MouseEvent) {
		if (isPanning) {
			offsetX = e.clientX - panStart.x;
			offsetY = e.clientY - panStart.y;
		}
		if (isDragging && selectedIds.size) {
			const { x, y } = toGrid(e.clientX, e.clientY);
			const dx = x - dragStartGrid.x,
				dy = y - dragStartGrid.y;
			objects = objects.map((o) => {
				if (!selectedIds.has(o.id) || o.locked) return o;
				const orig = dragOrigPos.get(o.id);
				if (!orig) return o;
				if (o.kind === 'text' && 'x' in orig) return { ...o, x: orig.x + dx, y: orig.y + dy };
				if (o.kind === 'line' && 'x1' in orig)
					return { ...o, x1: orig.x1 + dx, y1: orig.y1 + dy, x2: orig.x2 + dx, y2: orig.y2 + dy };
				return o;
			});
		}
	}
	function onCanvasMouseup() {
		if (isDragging && selectedIds.size) {
			// 이동이 실제로 일어난 경우 히스토리에 기록
			const moved = [...selectedIds].some((id) => {
				const orig = dragOrigPos.get(id);
				const cur = objects.find((o) => o.id === id);
				if (!orig || !cur) return false;
				if (cur.kind === 'text') return cur.x !== (orig as any).x || cur.y !== (orig as any).y;
				return true;
			});
			if (moved) {
				// 현재 상태를 히스토리에 밀어넣기 (snapshot은 드래그 시작 전에 찍었음)
			}
		}
		isPanning = false;
		isDragging = false;
	}
	function onWheel(e: WheelEvent) {
		e.preventDefault();
		const delta = e.deltaY > 0 ? 0.9 : 1.1;
		const newZoom = Math.max(0.1, Math.min(10, zoom * delta));
		// 커서 위치 기준 줌
		offsetX = e.clientX - (e.clientX - offsetX) * (newZoom / zoom);
		offsetY = e.clientY - (e.clientY - offsetY) * (newZoom / zoom);
		zoom = newZoom;
	}

	// ── 오브젝트 클릭 ────────────────────────────────────────────────────
	function onObjectMousedown(e: MouseEvent, id: string) {
		e.stopPropagation();
		if (e.button !== 0) return;

		const obj = objects.find((o) => o.id === id);
		if (!obj || obj.locked) return;

		if (e.shiftKey) {
			// Shift: 추가 선택
			const next = new Set(selectedIds);
			next.has(id) ? next.delete(id) : next.add(id);
			selectedIds = next;
		} else {
			if (!selectedIds.has(id)) selectedIds = new Set([id]);
		}

		// 입력창에 선택된 ID 채우기
		inputValue = [...selectedIds].join(' ');
		inputEl?.focus();

		// 드래그 준비
		snapshot();
		isDragging = true;
		dragStartGrid = toGrid(e.clientX, e.clientY);
		const entries: [string, OrigPos][] = objects
			.filter((o) => selectedIds.has(o.id))
			.map((o) =>
				o.kind === 'text'
					? [o.id, { x: o.x, y: o.y }]
					: [o.id, { x1: o.x1, y1: o.y1, x2: o.x2, y2: o.y2 }]
			);
		dragOrigPos = new Map(entries);
	}

	// ── 그리드 패턴 ──────────────────────────────────────────────────────
	let cellPx = $derived(GRID * zoom);
	let gridOffX = $derived(((offsetX % cellPx) + cellPx) % cellPx);
	let gridOffY = $derived(((offsetY % cellPx) + cellPx) % cellPx);

	// ── Canvas API 텍스트 너비 측정 ───────────────────────────────────────
	let _mc: HTMLCanvasElement | null = null;
	function measureLine(text: string, fontSize: number, bold: boolean, italic: boolean): number {
		if (typeof document === 'undefined') return text.length * fontSize * 0.6;
		if (!_mc) _mc = document.createElement('canvas');
		const ctx = _mc.getContext('2d')!;
		ctx.font = `${italic ? 'italic' : 'normal'} ${bold ? 'bold' : 'normal'} ${fontSize}px 'A2z', sans-serif`;
		return ctx.measureText(text).width;
	}

	// ── 뷰포트 크기 (좌표 레이블 범위 계산용) ─────────────────────────────
	// SSR 안전: $state 초기값은 0, 브라우저에서 $effect로 설정
	let viewW = $state(800);
	let viewH = $state(600);
	function onResize() {
		viewW = window.innerWidth;
		viewH = window.innerHeight - 56;
	}

	// 브라우저에서만 실행: 초기 offset (0,0을 화면 중앙으로), 뷰포트 크기
	$effect(() => {
		viewW = window.innerWidth;
		viewH = window.innerHeight - 56;
		offsetX = viewW / 2;
		offsetY = viewH / 2;
	});

	// ── 그리드 좌표 레이블: 줌에 따라 ~400px 간격 ────────────────────────
	function niceStep(raw: number): number {
		if (raw <= 1) return 1;
		const mag = Math.pow(10, Math.floor(Math.log10(raw)));
		const n = raw / mag;
		if (n < 1.5) return mag;
		if (n < 3.5) return 2 * mag;
		if (n < 7.5) return 5 * mag;
		return 10 * mag;
	}
	let labelStep = $derived(Math.max(1, niceStep(50 / cellPx)) * 2); // 점 2개 간격마다 레이블
	// 배경 그리드 선 간격: ~50px 기준 niceStep (레이블의 절반 밀도, 항상 정수)
	let gridStep = $derived(Math.max(1, niceStep(50 / cellPx)));
	let gridStepPx = $derived(cellPx * gridStep);
	let gridStepOffX = $derived(((offsetX % gridStepPx) + gridStepPx) % gridStepPx);
	let gridStepOffY = $derived(((offsetY % gridStepPx) + gridStepPx) % gridStepPx);

	let labelXs = $derived.by(() => {
		const step = labelStep,
			cp = cellPx;
		const xs: number[] = [];
		for (
			let g = Math.ceil((-offsetX / cp - step) / step) * step;
			g <= (viewW - offsetX) / cp + step;
			g += step
		)
			xs.push(g);
		return xs;
	});
	let labelYs = $derived.by(() => {
		const step = labelStep,
			cp = cellPx;
		const ys: number[] = [];
		for (
			let g = Math.ceil((-offsetY / cp - step) / step) * step;
			g <= (viewH - offsetY) / cp + step;
			g += step
		)
			ys.push(g);
		return ys;
	});

	// ── 더블클릭 → :r 자동완성 ───────────────────────────────────────────
	function onObjectDblclick(e: MouseEvent, id: string) {
		e.stopPropagation();
		const obj = objects.find((o) => o.id === id);
		if (!obj || obj.kind !== 'text') return;
		inputValue = `:r ${obj.id} ${obj.content}`;
		inputEl?.focus();
		setTimeout(() => inputEl?.setSelectionRange(`:r ${obj.id} `.length, inputValue.length), 0);
	}

	onMount(() => {
		const keydownHandler = window.addEventListener('keydown', (e) => {
			if (e.key === ':' && document.activeElement !== inputEl) {
				// ":" 키 → 입력창에 ":" 자동 입력
				e.preventDefault();
				inputValue = ':';
				inputEl?.focus();
			} else if (e.key === 'Enter' && document.activeElement !== inputEl) {
				// Enter 키 → 입력창에 현재 커서 좌표 자동 입력
				e.preventDefault();
				inputEl?.focus();
				inputEl?.setSelectionRange(0, 0);
			} else if (e.key === '/' && document.activeElement !== inputEl) {
				// "/" 키 → 입력창에 "/" 자동 입력 (검색)
				e.preventDefault();
				inputValue = '/';
				inputEl?.focus();
			}
		});

		return () => {
			window.removeEventListener('keydown', keydownHandler);
		};
	});
</script>

<svelte:window onresize={onResize} />

<!-- 캔버스 영역 -->
<div
	class="canvas-wrap"
	onmousedown={onCanvasMousedown}
	onmousemove={onCanvasMousemove}
	onmouseup={onCanvasMouseup}
	onwheel={onWheel}
	role="application"
	aria-label="Scribbler 캔버스"
>
	<svg class="canvas" width="100%" height="100%">
		<defs>
			<pattern
				id="grid-pat"
				width={gridStepPx}
				height={gridStepPx}
				patternUnits="userSpaceOnUse"
				x={gridStepOffX}
				y={gridStepOffY}
			>
				<circle cx="0" cy="0" r="1.8" fill="#8080b8" opacity="0.55" />
			</pattern>
		</defs>

		{#if showGrid}
			{#if gridStepPx >= 16}<rect width="100%" height="100%" fill="url(#grid-pat)" />{/if}
		{/if}

		<g transform="translate({offsetX},{offsetY}) scale({zoom})">
			{#if showGrid}
				<!-- 좌표 레이블 -->
				{#each labelXs as lgx}
					{#each labelYs as lgy}
						<text
							x={lgx * GRID}
							y={lgy * GRID - 5 / zoom}
							text-anchor="middle"
							fill={lgx === 0 && lgy === 0 ? '#5555aa' : '#7777aa'}
							opacity="0.75"
							font-size={9 / zoom}
							font-family="monospace"
							style="user-select:none; pointer-events:none">{lgx},{lgy}</text
						>
					{/each}
				{/each}
			{/if}

			{#each objects as obj (obj.id)}
				{@const sel = selectedIds.has(obj.id)}

				{#if obj.kind === 'text'}
					{@const px = gx(obj.x)}
					{@const py = gy(obj.y)}
					{@const lines = obj.content.split('\\\\')}
					{@const lineH = obj.fontSize * 1.4}
					{@const pad = 4}
					{@const bw =
						Math.max(...lines.map((l) => measureLine(l, obj.fontSize, obj.bold, obj.italic))) +
						pad * 2}
					{@const bh = (lines.length - 1) * lineH + obj.fontSize * 1.2 + pad * 2}
					{@const bx = px - pad}
					{@const by = py - obj.fontSize - pad}
					<!-- svelte-ignore a11y_no_static_element_interactions -->
					<g
						class="obj"
						class:selected={sel}
						class:locked={obj.locked}
						onmousedown={(e) => onObjectMousedown(e, obj.id)}
						ondblclick={(e) => onObjectDblclick(e, obj.id)}
						opacity={obj.alpha / 100}
					>
						<!-- 점선 보더 -->
						<rect
							x={bx}
							y={by}
							width={bw}
							height={bh}
							rx="0"
							ry="0"
							fill="none"
							stroke={sel ? '#4f8ef7' : obj.color}
							stroke-width={0.6 / zoom}
							stroke-dasharray={`${5 / zoom} ${5 / zoom}`}
							opacity={sel ? 0.6 : 0.15}
						/>
						<!-- 텍스트 (\\로 줄바꿈) -->
						<text
							x={px}
							y={py}
							fill={obj.color}
							font-size={obj.fontSize}
							font-family="'A2z', sans-serif"
							font-weight={obj.bold ? 'bold' : 'normal'}
							font-style={obj.italic ? 'italic' : 'normal'}
							text-decoration={obj.underline ? 'underline' : 'none'}
							style="user-select:none"
							>{#each lines as line, idx}<tspan x={px} dy={idx === 0 ? 0 : lineH}>{line}</tspan
								>{/each}</text
						>
						{#if sel}
							<rect
								x={bx - 1}
								y={by - 1}
								width={bw + 2}
								height={bh + 2}
								fill="none"
								stroke="#4f8ef7"
								stroke-width={1.5 / zoom}
								rx="7"
							/>
						{/if}
						{#if showIds}
							{@const _iw = (obj.id.length * 6.5) / zoom}
							{@const _ih = 11 / zoom}
							<rect
								x={bx - 1 / zoom}
								y={by - _ih - 1 / zoom}
								width={_iw + 4 / zoom}
								height={_ih}
								rx={2 / zoom}
								fill={sel ? '#4f8ef7' : '#d06820'}
								opacity="0.18"
								style="pointer-events:none"
							/>
							<text
								x={bx + 1 / zoom}
								y={by - 2 / zoom}
								fill={sel ? '#4f8ef7' : '#c05818'}
								font-size={8 / zoom}
								font-family="monospace"
								opacity="0.85"
								style="user-select:none">{obj.id}</text
							>
						{/if}
					</g>
				{:else if obj.kind === 'line'}
					{@const px1 = gx(obj.x1)}
					{@const py1 = gy(obj.y1)}
					{@const px2 = gx(obj.x2)}
					{@const py2 = gy(obj.y2)}
					{@const mx = (px1 + px2) / 2}
					{@const my = (py1 + py2) / 2}
					<!-- svelte-ignore a11y_no_static_element_interactions -->
					<g
						class="obj"
						class:selected={sel}
						class:locked={obj.locked}
						onmousedown={(e) => onObjectMousedown(e, obj.id)}
						opacity={obj.alpha / 100}
					>
						<!-- 클릭 영역 확보용 투명 선 -->
						<line
							x1={px1}
							y1={py1}
							x2={px2}
							y2={py2}
							stroke="transparent"
							stroke-width={Math.max(8, obj.strokeWidth * 2)}
						/>
						<line
							x1={px1}
							y1={py1}
							x2={px2}
							y2={py2}
							stroke={obj.color}
							stroke-width={obj.strokeWidth}
							stroke-dasharray={sel ? `${4 / zoom} ${2 / zoom}` : undefined}
						/>
						{#if showIds}
							{@const _iw = (obj.id.length * 6.5) / zoom}
							{@const _ih = 11 / zoom}
							<rect
								x={mx - _iw / 2 - 2 / zoom}
								y={my - 4 / zoom - _ih}
								width={_iw + 4 / zoom}
								height={_ih}
								rx={2 / zoom}
								fill={sel ? '#4f8ef7' : '#d06820'}
								opacity="0.18"
								style="pointer-events:none"
							/>
							<text
								x={mx}
								y={my - 5 / zoom}
								fill={sel ? '#4f8ef7' : '#c05818'}
								font-size={8 / zoom}
								font-family="monospace"
								text-anchor="middle"
								opacity="0.85"
								style="user-select:none">{obj.id}</text
							>
						{/if}
					</g>
				{/if}
			{/each}

			<!-- 텍스트 입력 커서 -->
			<g opacity="0.5" style="pointer-events:none">
				<line
					x1={cursorX * GRID - 6 / zoom}
					y1={cursorY * GRID}
					x2={cursorX * GRID + 6 / zoom}
					y2={cursorY * GRID}
					stroke="#4f8ef7"
					stroke-width={1.5 / zoom}
				/>
				<line
					x1={cursorX * GRID}
					y1={cursorY * GRID - 10 / zoom}
					x2={cursorX * GRID}
					y2={cursorY * GRID + 3 / zoom}
					stroke="#4f8ef7"
					stroke-width={1.5 / zoom}
				/>
			</g>
		</g>
	</svg>

	<!-- 줌 레벨 표시 -->
	<div class="zoom-badge">{Math.round(zoom * 100)}%</div>
</div>

<!-- 커맨드 입력창 -->
<div class="palette">
	<input
		bind:this={inputEl}
		bind:value={inputValue}
		class="palette-input"
		autocomplete="off"
		spellcheck={false}
		placeholder="텍스트 @x,y  또는  :커맨드  또는  /검색"
		autofocus
		onkeydown={onKeydown}
	/>
</div>

<style>
	.canvas-wrap {
		position: fixed;
		inset: 0;
		bottom: 56px;
		overflow: hidden;
		cursor: crosshair;
		user-select: none;
	}
	.canvas-wrap.panning {
		cursor: grab;
	}

	.obj {
		cursor: pointer;
	}
	.obj.locked {
		cursor: not-allowed;
	}

	.zoom-badge {
		position: absolute;
		bottom: 8px;
		right: 12px;
		font-size: 11px;
		color: #999;
		font-family: monospace;
		pointer-events: none;
	}

	.palette {
		position: fixed;
		bottom: 0;
		left: 0;
		right: 0;
		height: 56px;
		background: rgba(255, 255, 255, 0.96);
		border-top: 1px solid #ddd;
		display: flex;
		align-items: center;
		padding: 0 12px;
	}

	.palette-input {
		width: 100%;
		font-size: 14px;
		font-family: monospace;
		padding: 8px 12px;
		border: 1px solid #ccc;
		border-radius: 6px;
		outline: none;
		background: white;
	}
	.palette-input:focus {
		border-color: #4f8ef7;
	}
</style>
