<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import Two from "two.js";
  import type { Path } from "two.js/src/path";
  import type { Line as PathLine } from "two.js/src/shapes/line";
  import ControlTab from "./lib/ControlTab.svelte";
  import Navbar from "./lib/Navbar.svelte";
  import _ from "lodash";
  import {
    easeInOutQuad,
    getCurvePoint,
    getMousePos,
    getRandomColor,
    quadraticToCubic,
    radiansToDegrees,
    shortestRotation,
  } from "./utils";
  import hotkeys from 'hotkeys-js';
  import {ConstantHeadingSolver} from "./utils/ConstantHeadingSolver";
  import {CubicBezierCurveImpl} from "./utils/CubicBezierCurve.js";

  let two: Two;
  let twoElement: HTMLDivElement;

  let settings: FPASettings;

  let pointRadius = 1.15;
  let lineWidth = 0.57;
  let robotWidth = 16;
  let robotHeight = 16;

  let percent: number = 0;

  // Local storage keys
  const LOCAL_STORAGE_START_POINT_KEY = 'redform_trajectory_startPoint';
  const LOCAL_STORAGE_LINES_KEY = 'redform_trajectory_lines';



  /**
   * Converter for X axis from inches to pixels.
   */
  $: x = d3
    .scaleLinear()
    .domain([0, 144])
    .range([0, twoElement?.clientWidth ?? 144]);

  /**
   * Converter for Y axis from inches to pixels.
   */
  $: y = d3
    .scaleLinear()
    .domain([0, 144])
    .range([twoElement?.clientHeight ?? 144, 0]);

  let lineGroup = new Two.Group();
  lineGroup.id = "line-group";
  let pointGroup = new Two.Group();
  pointGroup.id = "point-group";
  let nameGroup = new Two.Group(); // Group for line names
  nameGroup.id = "name-group";

  // Reactive statements to save to localStorage on change
  $: {
    if (typeof localStorage !== 'undefined' && startPoint) {
      try {
        localStorage.setItem(LOCAL_STORAGE_START_POINT_KEY, JSON.stringify(startPoint));
      } catch (e) {
        console.error("Error saving startPoint to localStorage:", e);
      }
    }
  }

  $: {
    if (typeof localStorage !== 'undefined' && lines) {
      try {
        localStorage.setItem(LOCAL_STORAGE_LINES_KEY, JSON.stringify(lines));
      } catch (e) {
        console.error("Error saving lines to localStorage:", e);
      }
    }
  }

  // Default initial values for trajectory
  const defaultStartPointData: Point = {
    x: 8,
    y: 80,
    heading: "constant",
    degrees: 0,
  };

  const defaultLinesData: Line[] = [
    {
      endPoint: { x: 36, y: 80, heading: "constant", degrees: 0 },
      controlPoints: [],
      color: getRandomColor(), // getRandomColor is imported
    },
  ];

  let startPoint: Point = _.cloneDeep(defaultStartPointData); // lodash (_) is imported
  let lines: Line[] = _.cloneDeep(defaultLinesData);

  $: points = (() => {
    let _points = [];
    let startPointElem = new Two.Circle(
      x(startPoint.x),
      y(startPoint.y),
      x(pointRadius)
    );
    startPointElem.id = `point-0-0`;
    startPointElem.fill = lines[0].color;
    startPointElem.noStroke();

    _points.push(startPointElem);

    lines.forEach((line, idx) => {
      [line.endPoint, ...line.controlPoints].forEach((point, idx1) => {
        if (idx1 > 0) {
          let pointGroup = new Two.Group();
          pointGroup.id = `point-${idx + 1}-${idx1}`;

          let pointElem = new Two.Circle(
            x(point.x),
            y(point.y),
            x(pointRadius)
          );
          pointElem.id = `point-${idx + 1}-${idx1}-background`;
          pointElem.fill = line.color;
          pointElem.noStroke();

          let pointText = new Two.Text(
            `${idx1}`,
            x(point.x),
            y(point.y - 0.15),
            x(pointRadius)
          );
          pointText.id = `point-${idx + 1}-${idx1}-text`;
          pointText.size = x(1.55);
          pointText.leading = 1;
          pointText.family = "ui-sans-serif, system-ui, sans-serif";
          pointText.alignment = "center";
          pointText.baseline = "middle";
          pointText.fill = "white";
          pointText.noStroke();

          pointGroup.add(pointElem, pointText);
          _points.push(pointGroup);
        } else {
          let pointElem = new Two.Circle(
            x(point.x),
            y(point.y),
            x(pointRadius)
          );
          pointElem.id = `point-${idx + 1}-${idx1}`;
          pointElem.fill = line.color;
          pointElem.noStroke();
          _points.push(pointElem);
        }
      });
    });

    return _points;
  })();

  $: path = (() => {
    let _path: (Path | PathLine)[] = [];
    nameGroup.remove(nameGroup.children); // Clear old names before adding new ones

    lines.forEach((line, idx) => {
      let _startPoint = idx === 0 ? startPoint : lines[idx - 1].endPoint;

      let lineElem: Path | PathLine;
      let midPoint: BasePoint; // To position the name

      if (line.controlPoints.length > 2) {
        // Approximate an n-degree bezier curve by sampling it at 100 points
        const samples = 100;
        const cps = [_startPoint, ...line.controlPoints, line.endPoint];
        let points = [new Two.Anchor(x(_startPoint.x), y(_startPoint.y), 0, 0, 0, 0, Two.Commands.move)];
        for (let i = 1; i <= samples; ++i) {
          const point = getCurvePoint(i / samples, cps);
          points.push(new Two.Anchor(x(point.x), y(point.y), 0, 0, 0, 0, Two.Commands.line));
        }
        points.forEach((point) => (point.relative = false));

        lineElem = new Two.Path(points);
        lineElem.automatic = false;
        // Calculate midpoint for n-degree Bezier (approximate)
        const midSamplePoint = getCurvePoint(0.5, cps);
        midPoint = { x: x(midSamplePoint.x), y: y(midSamplePoint.y) };

      } else if (line.controlPoints.length > 0) {
        let cp1 = line.controlPoints[1]
          ? line.controlPoints[0]
          : quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
              .Q1;
        let cp2 =
          line.controlPoints[1] ??
          quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
            .Q2;

        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            x(_startPoint.x),
            y(_startPoint.y),
            x(cp1.x),
            y(cp1.y),
            Two.Commands.move
          ),
          new Two.Anchor(
            x(line.endPoint.x),
            y(line.endPoint.y),
            x(cp2.x),
            y(cp2.y),
            x(line.endPoint.x),
            y(line.endPoint.y),
            Two.Commands.curve
          ),
        ];
        points.forEach((point) => (point.relative = false));

        lineElem = new Two.Path(points);
        lineElem.automatic = false;
        // Calculate midpoint for cubic Bezier
        const t = 0.5;
        const mt = 1 - t;
        midPoint = {
          x: mt * mt * mt * x(_startPoint.x) + 3 * mt * mt * t * x(cp1.x) + 3 * mt * t * t * x(cp2.x) + t * t * t * x(line.endPoint.x),
          y: mt * mt * mt * y(_startPoint.y) + 3 * mt * mt * t * y(cp1.y) + 3 * mt * t * t * y(cp2.y) + t * t * t * y(line.endPoint.y)
        };
      } else {
        lineElem = new Two.Line(
          x(_startPoint.x),
          y(_startPoint.y),
          x(line.endPoint.x),
          y(line.endPoint.y)
        );
        // Calculate midpoint for straight line
        midPoint = {
          x: (x(_startPoint.x) + x(line.endPoint.x)) / 2,
          y: (y(_startPoint.y) + y(line.endPoint.y)) / 2
        };
      }

      lineElem.id = `line-${idx + 1}`;
      lineElem.stroke = line.color;
      lineElem.linewidth = x(lineWidth);
      lineElem.noFill();

      _path.push(lineElem);

      // Add name text if it exists
      if (line.name) {
        const nameText = new Two.Text(line.name, midPoint.x, midPoint.y - 10); // Position slightly above midpoint
        nameText.id = `name-${idx + 1}`;
        nameText.size = 16; // Increased font size
        nameText.family = 'Arial, sans-serif'; // Set font family
        nameText.weight = 500; // Set font weight
        nameText.fill = line.color;
        nameText.alignment = 'center';
        nameText.baseline = 'bottom';
        nameText.noStroke();
        nameGroup.add(nameText); // Add text to the name group instead of path array
      }
    });

    return _path;
  })();

  let robotXY: BasePoint = { x: 0, y: 0 };
  // Calculate robot heading (degrees)
  $: robotHeading = (((startPoint.degrees ?? 0) % 360) + 360) % 360;

  $: {
    let totalLineProgress = (lines.length * Math.min(percent, 99.999999999)) / 100;
    let currentLineIdx = Math.min(Math.trunc(totalLineProgress), lines.length - 1);
    let currentLine = lines[currentLineIdx];

    let linePercent = easeInOutQuad(totalLineProgress - Math.floor(totalLineProgress));
    let _startPoint = currentLineIdx === 0 ? startPoint : lines[currentLineIdx - 1].endPoint;
    let robotInchesXY = getCurvePoint(linePercent, [_startPoint, ...currentLine.controlPoints, currentLine.endPoint]);
    robotXY = { x: x(robotInchesXY.x), y: y(robotInchesXY.y) };

    if (x.invert(robotInchesXY.x) === startPoint.x && y.invert(robotInchesXY.y) === startPoint.y) {
      robotHeading = ((startPoint.degrees ?? 0) % 360 + 360) % 360;
    } else {
      switch (currentLine.endPoint.heading) {
        case "linear":
          robotHeading = ((-shortestRotation(currentLine.endPoint.startDeg, currentLine.endPoint.endDeg, linePercent) % 360) + 360) % 360;
          break;
        case "constant":
          robotHeading = ((-currentLine.endPoint.degrees % 360) + 360) % 360;
          break;
        case "tangential":
          const nextPointInches = getCurvePoint(
            linePercent + (currentLine.endPoint.reverse ? -0.01 : 0.01),
            [_startPoint, ...currentLine.controlPoints, currentLine.endPoint]
          );
          const nextPoint = { x: x(nextPointInches.x), y: y(nextPointInches.y) };

          const dx = nextPoint.x - robotXY.x;
          const dy = nextPoint.y - robotXY.y;

          if (dx !== 0 || dy !== 0) {
            const angle = Math.atan2(dy, dx);
            robotHeading = ((radiansToDegrees(angle) % 360) + 360) % 360;
          }
          break;
      }
    }
  }

  $: (() => {
    if (!two) {
      return;
    }

    two.renderer.domElement.style["z-index"] = "30";
    two.renderer.domElement.style["position"] = "absolute";
    two.renderer.domElement.style["top"] = "0px";
    two.renderer.domElement.style["left"] = "0px";
    two.renderer.domElement.style["width"] = "100%";
    two.renderer.domElement.style["height"] = "100%";

    two.clear();

    two.add(...path);
    two.add(...points);
    two.add(nameGroup); // Add the name group here

    two.update();
  })();

  let playing = false;

  let animationFrame: number;
  let startTime: number | null = null;
  let previousTime: number | null = null;

  function animate(timestamp: number) {
    if (!startTime) {
      startTime = timestamp;
    }

    if (previousTime !== null) {
      const deltaTime = timestamp - previousTime;

      if (percent >= 100) {
        percent = 0;
      } else {
        percent += (0.65 / lines.length) * (deltaTime * 0.1);
      }
    }

    previousTime = timestamp;

    if (playing) {
      requestAnimationFrame(animate);
    }
  }

  function play() {
    if (!playing) {
      playing = true;
      startTime = null;
      previousTime = null;
      animationFrame = requestAnimationFrame(animate);
    }
  }

  function pause() {
    playing = false;
    cancelAnimationFrame(animationFrame);
  }

  function fpa(fpaline: FPALine): Line {
    if(fpaline.heading === "constant") {
      const s = new ConstantHeadingSolver(settings, fpaline).getSolution().getPath();
      return new CubicBezierCurveImpl(s.getP0(), s.getP1(), s.getP2(), s.getP3()).exportAsLine(fpaline.endPoint.degrees ?? 0, fpaline.color);
    } else {
      return {
        endPoint: fpaline.endPoint,
        controlPoints: [],
        color: fpaline.color
      }
    }
  }

  onMount(() => {
    // Load trajectory from localStorage
    if (typeof localStorage !== 'undefined') {
      const savedStartPoint = localStorage.getItem(LOCAL_STORAGE_START_POINT_KEY);
      if (savedStartPoint) {
        try {
          const parsedStartPoint = JSON.parse(savedStartPoint);
          // Basic validation for startPoint structure
          if (typeof parsedStartPoint === 'object' && parsedStartPoint !== null && 'x' in parsedStartPoint && 'y' in parsedStartPoint && 'heading' in parsedStartPoint) {
            startPoint = parsedStartPoint;
          } else {
            console.error("Invalid startPoint data structure from localStorage. Using defaults.");
            localStorage.removeItem(LOCAL_STORAGE_START_POINT_KEY); // Clear invalid data
            startPoint = _.cloneDeep(defaultStartPointData); // Fallback to default
          }
        } catch (e) {
          console.error("Error parsing saved startPoint from localStorage:", e);
          localStorage.removeItem(LOCAL_STORAGE_START_POINT_KEY); // Clear corrupted data
          startPoint = _.cloneDeep(defaultStartPointData); // Fallback to default
        }
      }

      const savedLines = localStorage.getItem(LOCAL_STORAGE_LINES_KEY);
      if (savedLines) {
        try {
          const parsedLines = JSON.parse(savedLines);
          // Basic validation for lines structure
          if (Array.isArray(parsedLines) && parsedLines.every(line => typeof line === 'object' && line !== null && 'endPoint' in line && 'controlPoints' in line)) {
            lines = parsedLines;
            // Ensure lines have colors, as older saved data might not
            lines.forEach(line => {
              if (!line.color) {
                line.color = getRandomColor();
              }
            });
          } else {
            console.error("Invalid lines data structure from localStorage. Using defaults.");
            localStorage.removeItem(LOCAL_STORAGE_LINES_KEY); // Clear invalid data
            lines = _.cloneDeep(defaultLinesData); // Fallback to default
          }
        } catch (e) {
          console.error("Error parsing saved lines from localStorage:", e);
          localStorage.removeItem(LOCAL_STORAGE_LINES_KEY); // Clear corrupted data
          lines = _.cloneDeep(defaultLinesData); // Fallback to default
        }
      }
    } else {
      // Fallback to defaults if localStorage is not available
      startPoint = _.cloneDeep(defaultStartPointData);
      lines = _.cloneDeep(defaultLinesData);
    }

    two = new Two({
      fitted: true,
      type: Two.Types.svg,
    }).appendTo(twoElement);

    updateRobotImage();

    let currentElem: string | null = null;
    let isDown = false;

    two.renderer.domElement.addEventListener("mousemove", (evt: MouseEvent) => {
      const elem = document.elementFromPoint(evt.clientX, evt.clientY);
      if (isDown && currentElem) {
        const line = Number(currentElem.split("-")[1]) - 1;
        const point = Number(currentElem.split("-")[2]);

        const { x: xPos, y: yPos } = getMousePos(evt, two.renderer.domElement);

        const roundedX = Math.round(x.invert(xPos) * 10) / 10;
        const roundedY = Math.round(y.invert(yPos) * 10) / 10;

        if (line === -1) {
          startPoint.x = roundedX;
          startPoint.y = roundedY;
        } else {
          if (point === 0) {
            lines[line].endPoint.x = roundedX;
            lines[line].endPoint.y = roundedY;
          } else {
            lines[line].controlPoints[point - 1].x = roundedX;
            lines[line].controlPoints[point - 1].y = roundedY;
          }
        }
      } else {
        if (elem?.id.startsWith("point")) {
          two.renderer.domElement.style.cursor = "pointer";
          currentElem = elem.id;
        } else {
          two.renderer.domElement.style.cursor = "auto";
          currentElem = null;
        }
      }
    });
    two.renderer.domElement.addEventListener("mousedown", () => {
      isDown = true;
    });
    two.renderer.domElement.addEventListener("mouseup", () => {
      isDown = false;
    });
  });

  document.addEventListener("keydown", function (evt) {
    if (evt.code === "Space" && document.activeElement === document.body) {
      if (playing) {
        pause();
      } else {
        play();
      }
    }
  });

  function saveFile() {
    const jsonString = JSON.stringify({ startPoint, lines });

    const blob = new Blob([jsonString], { type: "application/json" });

    const linkObj = document.createElement("a");

    const url = URL.createObjectURL(blob);

    linkObj.href = url;
    linkObj.download = "trajectory.txt";

    document.body.appendChild(linkObj);

    linkObj.click();

    document.body.removeChild(linkObj);

    URL.revokeObjectURL(url);
  }

  function loadFile(evt: Event) {
    const elem = evt.target as HTMLInputElement;
    const file = elem.files?.[0];

    if (file) {
      const reader = new FileReader();

      reader.onload = function (e: ProgressEvent<FileReader>) {
        try {
          const result = e.target?.result as string;

          const jsonObj: {
            startPoint: Point;
            lines: Line[];
          } = JSON.parse(result);

          startPoint = jsonObj.startPoint;
          lines = jsonObj.lines;
        } catch (err) {
          console.error(err);
        }
      };

      reader.readAsText(file);
    }
  }

  function loadRobot(evt: Event) {
    const elem = evt.target as HTMLInputElement;
    const file = elem.files?.[0];

    if (file && file.type === "image/png") {
      const reader = new FileReader();

      reader.onload = function (e: ProgressEvent<FileReader>) {
        const result = e.target?.result as string;
        localStorage.setItem('robot.png', result);
        updateRobotImage();
      };

      reader.readAsDataURL(file);
    } else {
      console.error("Invalid file type. Please upload a PNG file.");
    }
  }

  function updateRobotImage() {
    const robotImage = document.querySelector('img[alt="Robot"]') as HTMLImageElement;
    const storedImage = localStorage.getItem('robot.png');
    if (robotImage && storedImage) {
      robotImage.src = storedImage;
    }
  }

  function addNewLine() {
  lines = [
    ...lines,
    {
      endPoint: {
        x: _.random(36, 108),
        y: _.random(36, 108),
        heading: "tangential",
        reverse: false,
      },
      controlPoints: [],
      color: getRandomColor(),
    },
  ];
}


function addControlPoint() {
  if (lines.length > 0) {
    const lastLine = lines[lines.length - 1];
    lastLine.controlPoints.push({
      x: _.random(36, 108),
      y: _.random(36, 108),
    });
  }
}

function removeControlPoint() {
  if (lines.length > 0) {
    const lastLine = lines[lines.length - 1];
    if (lastLine.controlPoints.length > 0) {
      lastLine.controlPoints.pop();
    }
  }
}

hotkeys('w', function(event, handler){
  event.preventDefault();
  addNewLine();
});


hotkeys('a', function(event, handler){
  event.preventDefault();
  addControlPoint();
  $: points;
  $: path;
  two.update();
});

hotkeys('s', function(event, handler){
  event.preventDefault();
  removeControlPoint();
  $: points;
  $: path;
  two.update();
});

</script>

<Navbar bind:lines bind:startPoint bind:settings bind:robotWidth bind:robotHeight {saveFile} {loadFile} {loadRobot}/>
<div
  class="w-screen h-screen pt-20 p-2 flex flex-row justify-center items-center gap-2"
>
  <div class="flex h-full justify-center items-center">
    <div
      bind:this={twoElement}
      class="h-full aspect-square rounded-lg shadow-md bg-neutral-50 dark:bg-neutral-900 relative overflow-clip"
    >
      <img
        src="/fields/intothedeep.webp"
        alt="Field"
        class="absolute top-0 left-0 w-full h-full rounded-lg z-10 pointer-events-none"
      />
      <img
        src={"/robot.png"}
        width={x(robotWidth)}
        height={x(robotHeight)}
        alt="Robot"
        style={`position: absolute; top: ${robotXY.y}px; left: ${robotXY.x}px; transform: translate(-50%, -50%) rotate(${robotHeading}deg); z-index: 20; width: ${x(robotWidth)}px; height: ${x(robotHeight)}px;`}
      />
    </div>
  </div>
  <ControlTab
    bind:robotHeading
    bind:playing
    {play}
    {pause}
    bind:startPoint
    bind:lines
    bind:robotWidth
    bind:robotHeight
    bind:percent
    bind:robotXY
    {x}
    {y}
    {fpa}
  />
</div>