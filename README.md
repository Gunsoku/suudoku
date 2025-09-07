<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>デジタル版「数独」</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body { font-family: sans-serif; text-align: center; margin:0; padding:0; }
    #sudoku { border-collapse: collapse; margin: 20px auto; }
    #sudoku td {
      width: 60px; height: 60px; text-align: center;
      border: 1px solid #999; font-size: 28px; position: relative;
    }
    #sudoku td input {
      width: 100%; height: 100%; text-align: center;
      font-size: 28px; border: none; outline: none;
      background: transparent; box-sizing: border-box;
    }
    /* 3x3ブロック線 */
    #sudoku td.block-right { border-right: 3px solid black !important; }
    #sudoku td.block-bottom { border-bottom: 3px solid black !important; }
    #sudoku td.block-left { border-left: 3px solid black !important; }
    #sudoku td.block-top { border-top: 3px solid black !important; }

    /* 二重枠セル */
    #sudoku td.double { border: 3px double red !important; }

    /* 外枠を太く */
    #sudoku tr:last-child td { border-bottom: 3px solid black !important; }
    #sudoku td:last-child { border-right: 3px solid black !important; }

    #timer { margin: 10px; font-weight: bold; font-size: 18px; }
    #message { margin: 10px; font-size: 18px; }
    #double-sum-input { width: 80px; font-size: 18px; text-align: center; padding:4px; }
    button { font-size: 16px; padding: 6px 12px; margin: 4px; }
  </style>
</head>
<body>
  <h1>デジタル版「数独」</h1>
  <div id="timer">経過時間: 0秒</div>
  <table id="sudoku"></table>
  <div style="margin:10px;">
    <label>二重枠合計: <input type="number" id="double-sum-input"></label>
    <button onclick="checkDoubleSum()">判定</button>
  </div>
  <div>
    <button onclick="showSolution()">解答を表示</button>
    <button onclick="newGame()">新しい問題</button>
  </div>
  <div id="message"></div>

<script>
let puzzle = [];
let solution = [];
let doubleCells = [];
let startTime;
let timerInterval;

const baseSolution = [
  [5,3,4,6,7,8,9,1,2],
  [6,7,2,1,9,5,3,4,8],
  [1,9,8,3,4,2,5,6,7],
  [8,5,9,7,6,1,4,2,3],
  [4,2,6,8,5,3,7,9,1],
  [7,1,3,9,2,4,8,5,6],
  [9,6,1,5,3,7,2,8,4],
  [2,8,7,4,1,9,6,3,5],
  [3,4,5,2,8,6,1,7,9]
];

function shuffleSolution() {
  let grid = JSON.parse(JSON.stringify(baseSolution));
  let nums = [1,2,3,4,5,6,7,8,9];
  nums.sort(()=>Math.random()-0.5);
  let map = {};
  for (let i=1;i<=9;i++) map[i] = nums[i-1];
  grid = grid.map(row => row.map(val => map[val]));

  for (let b=0;b<3;b++) {
    let rows = [0,1,2].map(i=>b*3+i);
    rows.sort(()=>Math.random()-0.5);
    let temp = [grid[rows[0]],grid[rows[1]],grid[rows[2]]];
    for (let i=0;i<3;i++) grid[b*3+i] = temp[i];
  }

  for (let b=0;b<3;b++) {
    let cols = [0,1,2].map(i=>b*3+i);
    cols.sort(()=>Math.random()-0.5);
    grid = grid.map(row=>{
      let temp = [row[cols[0]],row[cols[1]],row[cols[2]]];
      for (let i=0;i<3;i++) row[b*3+i]=temp[i];
      return row;
    });
  }

  let rowBlocks = [0,1,2]; rowBlocks.sort(()=>Math.random()-0.5);
  let newGrid = [];
  for (let b of rowBlocks) newGrid.push(...grid.slice(b*3,(b+1)*3));
  grid = newGrid;

  let colBlocks = [0,1,2]; colBlocks.sort(()=>Math.random()-0.5);
  grid = grid.map(row=>{
    let newRow=[];
    for (let b of colBlocks) newRow.push(...row.slice(b*3,(b+1)*3));
    return newRow;
  });

  return grid;
}

function generatePuzzle() {
  solution = shuffleSolution();
  puzzle = JSON.parse(JSON.stringify(solution));

  let blanks = 40;
  while (blanks > 0) {
    let r = Math.floor(Math.random()*9);
    let c = Math.floor(Math.random()*9);
    if (puzzle[r][c] !== 0) { puzzle[r][c] = 0; blanks--; }
  }

  doubleCells = [];
  while (doubleCells.length < 2) {
    let r = Math.floor(Math.random()*9);
    let c = Math.floor(Math.random()*9);
    let key = r + "," + c;
    if (puzzle[r][c] === 0 && !doubleCells.includes(key)) doubleCells.push(key);
  }
}

function drawBoard(showSol=false) {
  const table = document.getElementById("sudoku");
  table.innerHTML = "";
  for (let r=0;r<9;r++) {
    const row = document.createElement("tr");
    for (let c=0;c<9;c++) {
      const cell = document.createElement("td");
      if (c%3===2 && c!==8) cell.classList.add("block-right");
      if (r%3===2 && r!==8) cell.classList.add("block-bottom");
      if (c%3===0) cell.classList.add("block-left");
      if (r%3===0) cell.classList.add("block-top");
      if (doubleCells.includes(r+","+c)) cell.classList.add("double");

      const val = showSol ? solution[r][c] : puzzle[r][c];
      if (val !== 0) {
        cell.textContent = val;
      } else {
        const inp = document.createElement("input");
        inp.maxLength = 1;
        inp.oninput = () => {};
        cell.appendChild(inp);
      }
      row.appendChild(cell);
    }
    table.appendChild(row);
  }
}

function checkDoubleSum() {
  const inputVal = document.getElementById("double-sum-input").value;
  if (!inputVal) return;
  const sum = parseInt(inputVal);

  const correctSum = doubleCells.reduce((acc,key)=>{
    const [r,c] = key.split(",").map(Number);
    return acc + solution[r][c];
  },0);

  if (sum === correctSum) {
    clearInterval(timerInterval);
    document.getElementById("message").textContent = "お見事！クリアしました！ ";
  } else {
    document.getElementById("message").textContent = "不正解です！あともう一息！ファイト！";
  }
}

function showSolution() {
  drawBoard(true);
  clearInterval(timerInterval);
  document.getElementById("message").textContent = "解答を表示しました。";
}

function newGame() {
  generatePuzzle();
  drawBoard();
  document.getElementById("double-sum-input").value = "";
  startTime = Date.now();
  if (timerInterval) clearInterval(timerInterval);
  timerInterval = setInterval(()=>{
    let sec = Math.floor((Date.now()-startTime)/1000);
    document.getElementById("timer").textContent = "経過時間: " + sec + "秒";
  },1000);
  document.getElementById("message").textContent = "";
}

newGame();
</script>
</body>
</html>
