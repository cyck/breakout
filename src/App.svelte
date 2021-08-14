<script lang="ts">
  import { nanoid } from "nanoid";
  import { onMount } from "svelte";
  import Heart from "svelte-icons/fa/FaHeart.svelte";

  enum Playground {
    CEILING = 550,
    FLOOR = 0,
    LEFT = 0,
    RIGHT = 1000,
  }

  interface Size {
    width: number;
    height: number;
  }

  interface Object {
    id: string;
    x: number;
    y: number;
    size: Size;
    destroyed: boolean;
  }

  interface Brick extends Object {
    element: HTMLElement;
  }

  type Directions = [up: number, right: number];

  interface Bounce {
    (directions: Directions, element?: Object): Directions;
  }

  let game = {
    level: 1,
    lives: 3,
    score: 0,
    fps: 60,
    active: false,
    gameOver: false,
  };

  $: bricksPerRow = 6;
  $: bricksPerColumn = game.level + 2;
  $: bricksLength = bricksPerColumn * bricksPerRow;
  let engine: NodeJS.Timer = null;

  let ball: Object = {
    id: "ball",
    x: 0,
    y: 0,
    size: {
      width: 20,
      height: 20,
    },
    destroyed: false,
  };

  let paddle: Object = {
    id: "paddle",
    x: 200,
    y: 10,
    size: {
      width: 200,
      height: 20,
    },
    destroyed: false,
  };

  let bricks: Brick[] = [];

  const resetBallPosition = () => {
    ball.x = paddle.x + 90;
    ball.y = 32;
  };

  const initializeLevel = () => {
    const all = document.getElementsByClassName("brick");

    const nextBricks = [...all].map((el: HTMLDivElement) => {
      el.setAttribute("style", "background-color: rgb(60, 238, 43)");
      return {
        id: `brick-${nanoid()}`,
        x: el.offsetLeft,
        y: Playground.CEILING - el.offsetTop + el.clientHeight,
        size: {
          height: el.clientHeight,
          width: el.clientWidth,
        },
        destroyed: false,
        element: el,
      };
    });

    bricks = nextBricks;
  };

  const destroyBrick = (brick: Brick) => {
    brick.element.setAttribute("style", "background-color: transparent");
    brick.destroyed = true;
  };

  const movePaddle: svelte.JSX.MouseEventHandler<HTMLElement> = (e) => {
    const canMoveLeft = e.clientX >= Playground.LEFT + paddle.size.width / 2;
    const canMoveRight = e.clientX <= Playground.RIGHT - paddle.size.width / 2;
    if (canMoveLeft && canMoveRight) {
      paddle.x = e.offsetX - paddle.size.width / 2;
    }
  };

  const getBouncingAngle = (
    leftBoundary: number,
    rightBoundary: number,
    hit: number
  ) => {
    const length = rightBoundary - leftBoundary + 1;
    const ratio = hit / length;
    return Math.ceil(10 * ratio - 5);
  };

  const isCollision = (element: Object) => {
    const ballTop = ball.y + ball.size.height;
    const ballRight = ball.x + ball.size.width;
    const elementTop = element.y + element.size.height;
    const elementRight = element.x + element.size.width;

    const isOutside =
      ballTop < element.y ||
      ball.y > elementTop ||
      ballRight < element.x ||
      ball.x > elementRight;

    return !isOutside;
  };

  const bounceVerticalWalls: Bounce = ([up, right]) => {
    const isBouncingCeiling = ball.y > Playground.CEILING - ball.size.height;
    const isBouncingFloor = ball.y < Playground.FLOOR - ball.size.height;

    if (isBouncingCeiling || isBouncingFloor) {
      return [-up, right];
    }

    return [up, right];
  };

  const bounceHorizontalWalls: Bounce = ([up, right]) => {
    const isBouncingRight = ball.x + ball.size.width > Playground.RIGHT;
    const isBouncingLeft = ball.x < Playground.LEFT;

    if (isBouncingRight || isBouncingLeft) {
      return [up, -right];
    }

    return [up, right];
  };

  const bounceWalls: Bounce = ([up, right]) => {
    const [nextUp] = bounceVerticalWalls([up, right], ball);
    const [, nextRight] = bounceHorizontalWalls([up, right], ball);
    return [nextUp, nextRight];
  };

  const bouncePaddle: Bounce = ([up, right]) => {
    if (!isCollision(paddle)) return [up, right];

    const ballCenterX = ball.x + ball.size.width / 2;

    const angle = getBouncingAngle(
      paddle.x,
      paddle.x + paddle.size.width,
      ballCenterX - paddle.x
    );
    if (angle === 1) {
      return [-up, right / Math.abs(right)];
    } else {
      return [-up, angle];
    }
  };

  const bounceBrick: Bounce = ([up, right], brick: Brick) => {
    if (brick.destroyed) return [up, right];
    if (!isCollision(brick)) return [up, right];

    destroyBrick(brick);
    game.score++;
    return [-up, right];
  };

  const gameFinished = () => {
    const finished = bricks.every((brick) => brick.destroyed);
    if (!finished) return false;

    ++game.level;

    setTimeout(() => {
      startGame();
    }, 0);
    return true;
  };

  const gameOver = () => {
    const isBouncingFloor = ball.y < Playground.FLOOR - ball.size.height;
    if (!isBouncingFloor || isCollision(paddle)) return false;

    if (game.lives > 0) {
      game.lives--;
      return false;
    }

    game.gameOver = true;
    clearInterval(engine);
    engine = null;
    return true;
  };

  const restartGame = () => {
    game.gameOver = false;
    game.level = 1;
    game.lives = 3;
    game.score = 0;
    game.fps = 60;
    startGame();
  };

  const startGame = () => {
    let up = 8;
    let right = 1;
    clearInterval(engine);
    engine = null;
    resetBallPosition();
    initializeLevel();

    engine = setInterval(() => {
      bricks.forEach((brick) => {
        [up, right] = bounceBrick([up, right], brick);
      });

      [up, right] = bounceWalls([up, right]);
      [up, right] = bouncePaddle([up, right]);

      ball.y += up;
      ball.x += right;

      gameOver();
      gameFinished();
    }, 1000 / game.fps);
  };

  onMount(() => {
    if (game.gameOver) return;

    startGame();
  });
</script>

{#if !game.gameOver}
  <main on:mousemove|self={movePaddle}>
    <div class="statistics">
      <div class="lives-counter">
        {#each Array(game.lives) as _, i}
          <div class="life">
            <Heart />
          </div>
        {/each}
      </div>
      <div class="score">
        Score: {game.score}
      </div>
    </div>
    <div
      id="brick-panel"
      style="grid-template-columns: repeat({bricksPerColumn}, 1fr);"
    >
      {#each Array(bricksLength) as _, i}
        <div class="brick" />
      {/each}
    </div>

    <div
      id="ball"
      style="
          left:{ball.x}px; 
          bottom:{ball.y}px; 
          height:{ball.size.height}px; 
          width:{ball.size.width}px
        "
    />
    <div
      id="paddle"
      style="
          left:{paddle.x}px; 
          bottom:{paddle.y}px;
          height:{paddle.size.height}px; 
          width:{paddle.size.width}px;
        "
    />
  </main>
{:else}
  <main>
    <div class="game-over">
      <h1>Game Over</h1>
      <h6>Level: {game.level}</h6>
      <h6>Points: {game.score}</h6>
      <button class="game-over-button" on:click={restartGame}>Play again</button
      >
    </div>
  </main>
{/if}

<style>
  main {
    width: 1000px;
    height: 600px;
    margin: 100px auto 0 auto;
    position: relative;
    overflow: hidden;
    background-color: black;
    font-family: "Open Sans";
  }

  #brick-panel {
    padding: 0 80px;
    display: grid;
    gap: 5px;
  }

  .brick {
    height: 20px;
    border-radius: 10px;
  }

  #ball {
    border-radius: 50%;
    background-color: rgb(255, 96, 75);
    position: absolute;
  }

  #paddle {
    background-color: rgb(255, 255, 255);
    position: absolute;
    border-radius: 10px;
  }

  .statistics {
    display: flex;
    justify-content: space-between;
    padding: 5px;
    min-height: 52px;
  }

  .lives-counter {
    display: flex;
  }

  .life {
    color: red;
    width: 32px;
    height: 32px;
    margin-right: 5px;
  }

  .score {
    color: white;
  }

  h1 {
    color: white;
    font-size: 60px;
    text-align: center;
  }

  h6 {
    color: white;
    font-size: 20px;
    text-align: center;
  }

  .game-over {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
  }

  .game-over-button {
    margin-top: 1em;
    font-size: 20px;
    color: white;
    border: 1px white solid;
    border-radius: 1em;
    padding: 1em;
  }
</style>
