==> thread sweep on qwen2.5-1.5b-instruct-q4_k_m.gguf
    grid    : [1, 2, 6, 12, 16, 32]
    n_gpu   : 0
    physical: 12  logical: 16

   running: -m models\qwen2.5-1.5b-instruct-q4_k_m.gguf -t 1 -ngl 0 -p 0 -n 64 -r 2
   t=  1  tg128=   0.0 tok/s
   running: -m models\qwen2.5-1.5b-instruct-q4_k_m.gguf -t 2 -ngl 0 -p 0 -n 64 -r 2
   t=  2  tg128=   0.0 tok/s
   running: -m models\qwen2.5-1.5b-instruct-q4_k_m.gguf -t 6 -ngl 0 -p 0 -n 64 -r 2
   t=  6  tg128=   0.0 tok/s
   running: -m models\qwen2.5-1.5b-instruct-q4_k_m.gguf -t 12 -ngl 0 -p 0 -n 64 -r 2
   t= 12  tg128=   0.0 tok/s
   running: -m models\qwen2.5-1.5b-instruct-q4_k_m.gguf -t 16 -ngl 0 -p 0 -n 64 -r 2
   t= 16  tg128=   0.0 tok/s
   running: -m models\qwen2.5-1.5b-instruct-q4_k_m.gguf -t 32 -ngl 0 -p 0 -n 64 -r 2
   t= 32  tg128=   0.0 tok/s

# Bonus — Thread sweep

Model: `qwen2.5-1.5b-instruct-q4_k_m.gguf`  ·  GPU layers: `0`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 0.0 |
| 2 | 0.0 |
| 6 | 0.0 |
| 12 | 0.0 |
| 16 | 0.0 |
| 32 | 0.0 |

**Best**: `-t 1` at 0.0 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.