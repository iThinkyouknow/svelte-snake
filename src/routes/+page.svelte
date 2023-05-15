<script lang="ts">
    import { onMount } from "svelte";
    export const ssr = false;
    const score_base = 100;
    const width = 800;
    const height = width;
    const block_size = 10;
    let score = 0;
    let time_to_update = 300;
    let time_to_update_step = 0.9;
    let canvas: HTMLCanvasElement | null;
    let ctx: CanvasRenderingContext2D | null;
    let animation_frame = NaN;
    const DIRECTIONS = {
        UP: "UP",
        DOWN: "DOWN",
        LEFT: "LEFT",
        RIGHT: "RIGHT",
    } as const;
    type DIRECTIONS = (typeof DIRECTIONS)[keyof typeof DIRECTIONS];
    let snake_direction: DIRECTIONS = DIRECTIONS.RIGHT;

    const key_map: Record<string, DIRECTIONS> = {
        w: DIRECTIONS.UP,
        ArrowUp: DIRECTIONS.UP,
        a: DIRECTIONS.LEFT,
        ArrowLeft: DIRECTIONS.LEFT,
        s: DIRECTIONS.DOWN,
        ArrowDown: DIRECTIONS.DOWN,
        d: DIRECTIONS.RIGHT,
        ArrowRight: DIRECTIONS.RIGHT,
    } as const;

    const key_map_block: Record<DIRECTIONS, DIRECTIONS> = {
        [DIRECTIONS.UP]: DIRECTIONS.DOWN,
        [DIRECTIONS.DOWN]: DIRECTIONS.UP,
        [DIRECTIONS.LEFT]: DIRECTIONS.RIGHT,
        [DIRECTIONS.RIGHT]: DIRECTIONS.LEFT,
    } as const;

    function on_keydown(evt: KeyboardEvent) {
        const new_direction = key_map[evt.key];
        if (!new_direction || key_map_block[new_direction] === snake_direction)
            return;
        snake_direction = new_direction;
    }

    function round_to_block_size(num: number) {
        return Math.round(num / block_size) * block_size;
    }

    function init_ctx(
        ctx: CanvasRenderingContext2D,
        width: number,
        height: number
    ) {
        ctx.fillStyle = "black";
        ctx?.fillRect(0, 0, width, height);
    }

    type position = {
        x: number;
        y: number;
    };

    function make_position(
        x = round_to_block_size(width / 2),
        y = round_to_block_size(height / 2)
    ) {
        return {
            x,
            y,
        };
    }

    let snake: position[] = [make_position()];

    function make_random_position(random = Math.random) {
        return make_position(
            round_to_block_size(random() * (width - block_size)),
            round_to_block_size(random() * (height - block_size))
        );
    }

    let apple: position = make_random_position();

    let did_eat_apple = false;

    function draw_apple(ctx: CanvasRenderingContext2D, apple: position) {
        ctx.beginPath();
        ctx.fillStyle = "red";
        ctx.rect(apple.x, apple.y, block_size, block_size);
        ctx.fill();
        ctx.closePath();
    }

    function draw_snake(ctx: CanvasRenderingContext2D, snake: position[]) {
        ctx.beginPath();
        ctx.fillStyle = "limegreen";
        snake.forEach((part) => {
            ctx.rect(part.x, part.y, block_size, block_size);
        });
        ctx.fill();
        ctx.closePath();
    }

    function get_new_head_position(head: position, direction: DIRECTIONS) {
        let x = head.x;
        let y = head.y;
        if (direction === DIRECTIONS.UP) {
            y = Math.abs((y - block_size + height) % height);
        }
        if (direction === DIRECTIONS.DOWN) {
            y = Math.abs((y + block_size) % height);
        }
        if (direction === DIRECTIONS.LEFT) {
            x = Math.abs((x - block_size + width) % width);
        }
        if (direction === DIRECTIONS.RIGHT) {
            x = Math.abs((x + block_size) % width);
        }

        return { x, y };
    }

    function move_snake(snake: position[], direction: DIRECTIONS) {
        const head = snake.at(-1);
        if (!head) return snake;
        for (let i = 0; i < snake.length - 1; i++) {
            snake[i] = { ...snake[i + 1] };
        }

        const { x, y } = get_new_head_position(head, direction);
        head.x = x;
        head.y = y;

        return snake;
    }

    function if_snake_eats_apple(head: position, apple: position) {
        const keys: (keyof position)[] = ["x", "y"];
        return keys.every((pos) => head[pos] === apple[pos]);
    }

    function if_bang_into_self(snake: position[]) {
        const head = snake.at(-1);
        if (!head) return false;
        return snake
            .slice(0, -1)
            .some((part) => part.x === head.x && part.y === head.y);
    }

    let current_time = 0;
    function loop(ctx: CanvasRenderingContext2D) {
        animation_frame = requestAnimationFrame((time) => {
            let snake_head = snake.at(-1);
            if (time - current_time < time_to_update || !snake_head) {
                loop(ctx);
                return;
            }
            current_time = time;

            init_ctx(ctx, width, height);
            draw_apple(ctx, apple);
            if (did_eat_apple) {
                did_eat_apple = false;
                snake_head = get_new_head_position(snake_head, snake_direction);
                snake.push(snake_head);
            } else {
                snake = move_snake(snake, snake_direction);
            }

            if (if_snake_eats_apple(snake_head, apple)) {
                did_eat_apple = true;
                score += score_base;
                time_to_update *= time_to_update_step;
                apple = make_random_position();
            }

            if (if_bang_into_self(snake)) {
                score = 0;
                did_eat_apple = false;
                snake = [make_random_position()];
            }
            draw_snake(ctx, snake);
            loop(ctx);
        });
    }

    onMount(() => {
        if (canvas) {
            ctx = canvas.getContext("2d");
            if (ctx) {
                loop(ctx);
            }
        }
    });
</script>

<h1>Snake Svelte (Cause I am bored)</h1>
<canvas bind:this={canvas} id="canvas" {width} {height} />
<h1>Score: {score}</h1>
<svelte:window on:keydown={on_keydown} />

<style>
    canvas {
        border: 1px solid red;
    }
</style>
