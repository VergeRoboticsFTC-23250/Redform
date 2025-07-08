<script lang="ts">
  import prettier from "prettier";
  import prettierJavaPlugin from "prettier-plugin-java";
  import { onMount } from "svelte";
  import { copy } from "svelte-copy";
  import Highlight from "svelte-highlight";
  import { java } from "svelte-highlight/languages";
  import codeStyle from "svelte-highlight/styles/androidstudio";
  import { cubicInOut } from "svelte/easing";
  import { fade, fly } from "svelte/transition";
  import { darkMode } from "../stores";
  import { getRandomColor, titleCase } from "../utils";

  export let saveFile: () => any;
  export let loadFile: (evt: any) => any;
  export let loadRobot: (evt: any) => any;

  export let robotWidth: number;
  export let robotHeight: number;

  export let startPoint: Point;
  export let lines: Line[];
  export let settings: FPASettings;
  
  settings = {
    vMax: 50,
    mass: 10,
    kFriction: 0.01,
    drag: 0.01,
    aVel: 3,
    bTolerance: 1,
    sTolerance: 1,
    rWidth: 16,
    rHeight: 16
  }

  let dialogOpen = false;
  let settingsOpen = false;
  let resetDialogOpen = false;

  onMount(() => {
    darkMode.subscribe((val) => {
      if (val === "light") {
        document.documentElement.classList.remove("dark");
      } else {
        document.documentElement.classList.add("dark");
      }
    });

    window.onbeforeunload = () => {
      return "Are you sure you want to leave?";
    };
  });

  let exportedCode = "";

  let coordSystem = "pedro";

  function convertPedroToFTC(pose: { x: number; y: number; degrees: number }) {
    // Normalize: subtract 72 from x and y
    const normX = pose.x - 72;
    const normY = pose.y - 72;
    const headingRad = pose.degrees * Math.PI / 180;
    // Rotate by -Math.PI/2
    const x = normX * Math.cos(-Math.PI/2) - normY * Math.sin(-Math.PI/2);
    const y = normX * Math.sin(-Math.PI/2) + normY * Math.cos(-Math.PI/2);
    const heading = headingRad - Math.PI/2;
    return { x, y, heading };
  }

  async function exportToCode() {
    if (coordSystem === "ftc") {
      let pose2dDecls = ["// Poses"];
      // Start pose
      pose2dDecls.push(`public static Pose2d startPose = new Pose2d(${(startPoint.x - 72).toFixed(3)}, ${(startPoint.y - 72).toFixed(3)}, Math.toRadians(${startPoint.degrees}));`);
      lines.forEach((line, idx) => {
        const name = line.name && line.name.trim() !== "" ? line.name.replace(/\s+/g, "_") : `path${idx+1}`;
        let headingExpr = "0";
        if (line.endPoint.heading === "constant" && typeof line.endPoint.degrees === 'number') {
          headingExpr = `Math.toRadians(${line.endPoint.degrees})`;
        } else if (line.endPoint.heading === "linear" && typeof line.endPoint.endDeg === 'number') {
          headingExpr = `Math.toRadians(${line.endPoint.endDeg})`;
        } else if (line.endPoint.heading === "tangential") {
          headingExpr = `0 /* WARNING: tangential heading not implemented */`;
        }
        pose2dDecls.push(`public static Pose2d ${name}Pose = new Pose2d(${(line.endPoint.x - 72).toFixed(3)}, ${(line.endPoint.y - 72).toFixed(3)}, ${headingExpr});`);
      });
      exportedCode = pose2dDecls.join("\n");
      dialogOpen = true;
      return;
    }
    const headingTypeToFunctionName = {
      constant: "setConstantHeadingInterpolation",
      linear: "setLinearHeadingInterpolation",
      tangential: "setTangentHeadingInterpolation",
    };

    const toJavaVarName = (name: string | undefined, defaultPrefix: string, index: number | string) => {
      let baseName = name;
      // Ensure baseName is a string before trying to replace spaces or test regex
      if (typeof baseName === 'string' && baseName.trim() !== '') {
        baseName = baseName.replace(/\s+/g, '_'); // Replace all whitespace sequences with a single underscore
        if (/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(baseName) && baseName !== "_") {
          // Basic check for Java keywords (add more if needed)
          const keywords = new Set(["abstract", "continue", "for", "new", "switch", "assert", "default", "goto", "package", "synchronized", "boolean", "do", "if", "private", "this", "break", "double", "implements", "protected", "throw", "byte", "else", "import", "public", "throws", "case", "enum", "instanceof", "return", "transient", "catch", "extends", "int", "short", "try", "char", "final", "interface", "static", "void", "class", "finally", "long", "strictfp", "volatile", "const", "float", "native", "super", "while"]);
          if (!keywords.has(baseName)) {
            return baseName;
          }
        }
      }
      return `${defaultPrefix}${index}`;
    };

    let poseFieldDeclarations: string[] = [];
    let pathFieldDeclarations: string[] = [];

    // Start Pose
    const startPoseVarName = "startPose";
    let startPointInitialHeadingRad = `Math.toRadians(${startPoint.degrees})`;
    poseFieldDeclarations.push(`public static Pose ${startPoseVarName} = new Pose(${startPoint.x.toFixed(3)}, ${startPoint.y.toFixed(3)}, ${startPointInitialHeadingRad});`);

    let previousPoseVarName = startPoseVarName;
    let firstTangent = false;
    lines.forEach((line, idx) => {
      const lineNameJava = toJavaVarName(line.name, "path", idx + 1);
      const endPoseVarName = `${lineNameJava}Pose`;
      let endPointHeadingRad = "0.0";
      if (line.endPoint.heading === "constant" && typeof line.endPoint.degrees === 'number') {
        endPointHeadingRad = `Math.toRadians(${line.endPoint.degrees.toFixed(3)})`;
      } else if (line.endPoint.heading === "linear" && typeof line.endPoint.endDeg === 'number') {
        endPointHeadingRad = `Math.toRadians(${line.endPoint.endDeg.toFixed(3)})`;
      } 
      if (endPointHeadingRad === "0.0" && !firstTangent) {
        firstTangent = true;
        poseFieldDeclarations.push(
        `public static Pose ${endPoseVarName} = new Pose(${line.endPoint.x.toFixed(3)}, ${line.endPoint.y.toFixed(3)}, ${endPointHeadingRad}); // tangential heading calculated in initPaths`
      );
      } else {
        poseFieldDeclarations.push(
        `public static Pose ${endPoseVarName} = new Pose(${line.endPoint.x.toFixed(3)}, ${line.endPoint.y.toFixed(3)}, ${endPointHeadingRad});`
      );
      }
      

      const pathSegmentVarName = lineNameJava;
      let pathType = "BezierLine";
      let pathInit = "";
      if (line.controlPoints.length > 0) {
        pathType = "BezierCurve";
        // For BezierCurve, all arguments must be Point objects
        let points = [
          `new Point(${previousPoseVarName}.getX(), ${previousPoseVarName}.getY(), Point.CARTESIAN)`
        ];
        points = points.concat(
          line.controlPoints.map(p => `new Point(${p.x.toFixed(3)}, ${p.y.toFixed(3)}, Point.CARTESIAN)`)
        );
        points.push(`new Point(${endPoseVarName}.getX(), ${endPoseVarName}.getY(), Point.CARTESIAN)`);
        pathInit = `public static Path ${pathSegmentVarName} = new Path(\n    new ${pathType}(\n        ${points.join(",\n        ")}\n    )\n);`;
      } else {
        // For BezierLine, use pose variables directly
        pathInit = `public static Path ${pathSegmentVarName} = new Path(\n    new ${pathType}(\n        ${previousPoseVarName},\n        ${endPoseVarName}\n    )\n);`;
      }
      pathFieldDeclarations.push(pathInit);
      previousPoseVarName = endPoseVarName;
    });

    let constructorOps: string[] = [];
    lines.forEach((line, idx) => {
      const lineNameJava = toJavaVarName(line.name, "path", idx + 1);
      const endPoseVarName = `${lineNameJava}Pose`;
      const prevPoseVarName = idx === 0 ? startPoseVarName : toJavaVarName(lines[idx-1].name, "path", idx) + "Pose";

      const { heading, degrees, startDeg, endDeg, reverse } = line.endPoint;
      let headingInterpolationSetup = "";

      if (heading === "constant") {
        headingInterpolationSetup = `${lineNameJava}.${headingTypeToFunctionName[heading]}(${endPoseVarName}.getHeading());`;
      } else if (heading === "linear") {
        headingInterpolationSetup = `${lineNameJava}.${headingTypeToFunctionName[heading]}(${prevPoseVarName}.getHeading(), ${endPoseVarName}.getHeading());`;
      } else if (heading === "tangential") {
        headingInterpolationSetup = `${lineNameJava}.${headingTypeToFunctionName[heading]}();`
        // Update the pose heading after setting tangential interpolation
        if (reverse) {
          headingInterpolationSetup += `\n${lineNameJava}.setReversed(true);`;
        }
        headingInterpolationSetup += `\n${endPoseVarName}.setHeading(${lineNameJava}.getEndTangent().getTheta());`;
      }
      if (headingInterpolationSetup) {
        constructorOps.push(headingInterpolationSetup + "\n");
      }
    });

    let fileContent = `// Poses
${poseFieldDeclarations.join("\n")}

// Paths
${pathFieldDeclarations.join("\n")}

// TODO run in opmode init
public static void initPaths() {
${constructorOps.map(op => `    ${op.replace(/\n/g, '\n    ')}`).join("\n")}
}
`;

    exportedCode = fileContent;

    dialogOpen = true;
  }

  function openSettings() {
    settingsOpen = true;
  }

</script>

<svelte:head>
  {@html codeStyle}
</svelte:head>

<div
  class="absolute top-0 left-0 w-full bg-neutral-50 dark:bg-neutral-900 shadow-md flex flex-row justify-between items-center px-6 py-4 border-b-[0.75px] border-[#e33a46]"
>
  <div class="flex flex-row justify-start items-center gap-2">
    <div class="font-semibold flex flex-col justify-start items-start">
      <div>Redform</div>

    </div>
    <a
      target="_blank"
      rel="norefferer"
      title="GitHub Repo"
      href="https://github.com/VergeRoboticsFTC-23250/Redform"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 30 30"
        class="size-8 dark:fill-white"
      >
        <path
          d="M15,3C8.373,3,3,8.373,3,15c0,5.623,3.872,10.328,9.092,11.63C12.036,26.468,12,26.28,12,26.047v-2.051 c-0.487,0-1.303,0-1.508,0c-0.821,0-1.551-0.353-1.905-1.009c-0.393-0.729-0.461-1.844-1.435-2.526 c-0.289-0.227-0.069-0.486,0.264-0.451c0.615,0.174,1.125,0.596,1.605,1.222c0.478,0.627,0.703,0.769,1.596,0.769 c0.433,0,1.081-0.025,1.691-0.121c0.328-0.833,0.895-1.6,1.588-1.962c-3.996-0.411-5.903-2.399-5.903-5.098 c0-1.162,0.495-2.286,1.336-3.233C9.053,10.647,8.706,8.73,9.435,8c1.798,0,2.885,1.166,3.146,1.481C13.477,9.174,14.461,9,15.495,9 c1.036,0,2.024,0.174,2.922,0.483C18.675,9.17,19.763,8,21.565,8c0.732,0.731,0.381,2.656,0.102,3.594 c0.836,0.945,1.328,2.066,1.328,3.226c0,2.697-1.904,4.684-5.894,5.097C18.199,20.49,19,22.1,19,23.313v2.734 c0,0.104-0.023,0.179-0.035,0.268C23.641,24.676,27,20.236,27,15C27,8.373,21.627,3,15,3z"
        ></path>
      </svg>
    </a>
  </div>
  <div class="flex flex-row justify-end items-end gap-3">
    <button title="Save trajectory as a file" on:click={() => saveFile()}>
      <svg
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
        stroke-width="2"
        stroke="currentColor"
        class="size-6"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="M3 16.5v2.25A2.25 2.25 0 0 0 5.25 21h13.5A2.25 2.25 0 0 0 21 18.75V16.5M16.5 12 12 16.5m0 0L7.5 12m4.5 4.5V3"
        />
      </svg>
    </button>
    <input
      id="file-input"
      type="file"
      accept=".txt"
      on:change={loadFile}
      class="hidden"
    />
    <label
      for="file-input"
      title="Load trajectory from a file"
      class="cursor-pointer"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
        stroke-width="2"
        stroke="currentColor"
        class="size-6"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="M3 16.5v2.25A2.25 2.25 0 0 0 5.25 21h13.5A2.25 2.25 0 0 0 21 18.75V16.5m-13.5-9L12 3m0 0 4.5 4.5M12 3v13.5"
        />
      </svg>
    </label>
    <button
      title="Delete/Reset path"
      on:click={() => {
        resetDialogOpen = true;
      }}
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
        stroke-width="2"
        stroke="currentColor"
        class="size-6"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="m14.74 9-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 0 1-2.244 2.077H8.084a2.25 2.25 0 0 1-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 0 0-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 0 1 3.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 0 0-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 0 0-7.5 0"
        />
      </svg>
    </button>
    <button title="Export path to code" on:click={exportToCode}>
      <svg
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
        stroke-width="2"
        stroke="currentColor"
        class="size-6"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="M17.25 6.75 22.5 12l-5.25 5.25m-10.5 0L1.5 12l5.25-5.25m7.5-3-4.5 16.5"
        />
      </svg>
    </button>
    <button title="Open Settings" on:click={openSettings}>
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="3"></circle><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path></svg>
    </button>
    <button
      title="Toggle Dark/Light Mode"
      on:click={() => {
        darkMode.toggle();
      }}
    >
      {#if $darkMode === "light"}
        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="2"
          stroke="currentColor"
          class="size-6"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M21.752 15.002A9.72 9.72 0 0 1 18 15.75c-5.385 0-9.75-4.365-9.75-9.75 0-1.33.266-2.597.748-3.752A9.753 9.753 0 0 0 3 11.25C3 16.635 7.365 21 12.75 21a9.753 9.753 0 0 0 9.002-5.998Z"
          />
        </svg>
      {:else}
        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="2"
          stroke="currentColor"
          class="size-6"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M12 3v2.25m6.364.386-1.591 1.591M21 12h-2.25m-.386 6.364-1.591-1.591M12 18.75V21m-4.773-4.227-1.591 1.591M5.25 12H3m4.227-4.773L5.636 5.636M15.75 12a3.75 3.75 0 1 1-7.5 0 3.75 3.75 0 0 1 7.5 0Z"
          />
        </svg>
      {/if}
    </button>
    <button>
    <input
            id="robot-input"
            type="file"
            accept="image/png"
            on:change={loadRobot}
            class="hidden"
    />
    <label
            for="robot-input"
            title="Load Robot Picture from a file"
            class="cursor-pointer"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 14.66V20a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h5.34"></path><polygon points="18 2 22 6 12 16 8 16 8 12 18 2"></polygon></svg>
    </label>
    </button>
  </div>
</div>

{#if resetDialogOpen}
  <div
    transition:fade={{ duration: 500, easing: cubicInOut }}
    class="bg-black bg-opacity-25 flex flex-col justify-center items-center absolute top-0 left-0 w-full h-full z-[1006]"
  >
    <div
      transition:fly={{ duration: 500, easing: cubicInOut, y: 20 }}
      class="flex flex-col justify-start items-start p-4 bg-white dark:bg-neutral-900 rounded-lg w-full max-w-md gap-3"
    >
      <p class="text-lg font-medium text-neutral-800 dark:text-neutral-200">Confirm Reset</p>
      <p class="text-sm text-neutral-600 dark:text-neutral-400">
        Are you sure you want to reset the path? This action cannot be undone.
      </p>
      <div class="flex flex-row justify-end items-center w-full gap-2 mt-2">
        <button
          class="px-4 py-2 text-sm font-medium text-neutral-700 dark:text-neutral-300 bg-neutral-200 dark:bg-neutral-700 hover:bg-neutral-300 dark:hover:bg-neutral-600 rounded-md transition-colors"
          on:click={() => {
            resetDialogOpen = false;
          }}
        >
          Cancel
        </button>
        <button
          class="px-4 py-2 text-sm font-medium text-white bg-red-500 hover:bg-red-600 rounded-md transition-colors"
          on:click={() => {
            startPoint = {
              x: 8,
              y: 80,
              heading: "linear",
              startDeg: 0,
              endDeg: 0
            };
            lines = [
              {
                endPoint: { x: 36, y: 80, heading: "linear", startDeg: 0, endDeg: 0 },
                controlPoints: [],
                color: getRandomColor(),
              },
            ];
            resetDialogOpen = false;
          }}
        >
          Reset Path
        </button>
      </div>
    </div>
  </div>
{/if}

{#if dialogOpen}
  <div
    transition:fade={{ duration: 500, easing: cubicInOut }}
    class="bg-black bg-opacity-25 flex flex-col justify-center items-center absolute top-0 left-0 w-full h-full z-[1005]"
  >
    <div
      transition:fly={{ duration: 500, easing: cubicInOut, y: 20 }}
      class="flex flex-col justify-start items-start p-4 bg-white dark:bg-neutral-900 rounded-lg w-full max-w-4xl gap-2.5"
    >
      <div class="flex flex-row justify-between items-center w-full">
        
        
      </div>

      <div class="relative w-full flex flex-col gap-3">
        <div class="flex flex-row justify-between items-center w-full mb-2">
          <div class="flex flex-row items-center gap-2">
            <p class="text-sm font-light text-neutral-700 dark:text-neutral-400">Here is the generated code:</p>
          </div>
          <div class="flex flex-row items-center gap-3">
            <label for="coord-system" class="text-sm font-light text-neutral-700 dark:text-neutral-300">Coordinate System:</label>
            <select id="coord-system" class="rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none px-2 py-1 text-sm" bind:value={coordSystem} on:change={exportToCode}>
              <option value="pedro">Pedro</option>
              <option value="ftc">FTC Standard</option>
            </select>
            <button
              class="ml-2"
              on:click={() => {
                dialogOpen = false;
              }}
              title="Close"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                stroke-width="2"
                stroke="currentColor"
                class="size-6 text-neutral-700 dark:text-neutral-400"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M6 18 18 6M6 6l12 12"
                />
              </svg>
            </button>
          </div>
        </div>
        <div class="relative w-full">
          <Highlight language={java} code={exportedCode} class="w-full" style="max-height:60vh;overflow:hidden;" />
          <button
            title="Copy code to clipboard"
            use:copy={exportedCode}
            class="absolute bottom-2 right-2 opacity-45 hover:opacity-100 transition-all duration-200 z-10"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="size-6"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M8.25 7.5V6.108c0-1.135.845-2.098 1.976-2.192.373-.03.748-.057 1.123-.08M15.75 18H18a2.25 2.25 0 0 0 2.25-2.25V6.108c0-1.135-.845-2.098-1.976-2.192a48.424 48.424 0 0 0-1.123-.08M15.75 18.75v-1.875a3.375 3.375 0 0 0-3.375-3.375h-1.5a1.125 1.125 0 0 1-1.125-1.125v-1.5A3.375 3.375 0 0 0 6.375 7.5H5.25m11.9-3.664A2.251 2.251 0 0 0 15 2.25h-1.5a2.251 2.251 0 0 0-2.15 1.586m5.8 0c.065.21.1.433.1.664v.75h-6V4.5c0-.231.035-.454.1-.664M6.75 7.5H4.875c-.621 0-1.125.504-1.125 1.125v12c0 .621.504 1.125 1.125 1.125h9.75c.621 0 1.125-.504 1.125-1.125V16.5a9 9 0 0 0-9-9Z"
              />
            </svg>
          </button>
        </div>
        <button
          title="Copy code to clipboard"
          use:copy={exportedCode}
          class="absolute bottom-2 right-2 opacity-45 hover:opacity-100 transition-all duration-200"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke-width="1.5"
            stroke="currentColor"
            class="size-6"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M8.25 7.5V6.108c0-1.135.845-2.098 1.976-2.192.373-.03.748-.057 1.123-.08M15.75 18H18a2.25 2.25 0 0 0 2.25-2.25V6.108c0-1.135-.845-2.098-1.976-2.192a48.424 48.424 0 0 0-1.123-.08M15.75 18.75v-1.875a3.375 3.375 0 0 0-3.375-3.375h-1.5a1.125 1.125 0 0 1-1.125-1.125v-1.5A3.375 3.375 0 0 0 6.375 7.5H5.25m11.9-3.664A2.251 2.251 0 0 0 15 2.25h-1.5a2.251 2.251 0 0 0-2.15 1.586m5.8 0c.065.21.1.433.1.664v.75h-6V4.5c0-.231.035-.454.1-.664M6.75 7.5H4.875c-.621 0-1.125.504-1.125 1.125v12c0 .621.504 1.125 1.125 1.125h9.75c.621 0 1.125-.504 1.125-1.125V16.5a9 9 0 0 0-9-9Z"
            />
          </svg>
        </button>
      </div>
    </div>
  </div>
{/if}

{#if settingsOpen}
  <div
          transition:fade={{ duration: 500, easing: cubicInOut }}
          class="bg-black bg-opacity-25 flex flex-col justify-center items-center absolute top-0 left-0 w-full h-full z-[1005]"
  >
    <div
            transition:fly={{ duration: 500, easing: cubicInOut, y: 20 }}
            class="flex flex-col justify-start items-start p-4 bg-white dark:bg-neutral-900 rounded-lg w-full max-w-4xl gap-2.5"
    >
      <div class="flex flex-row justify-between items-center w-full">
        <p class="text-sm font-light text-neutral-700 dark:text-neutral-400">
          Settings:
        </p>
        <button
                class=""
                on:click={() => {
                settingsOpen = false;
                console.log(settings.vMax, settings.mass, settings.kFriction, settings.drag, settings.aVel, settings.bTolerance, settings.sTolerance, settings.rWidth, settings.rHeight);
          }}
        ><svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                stroke-width="2"
                stroke="currentColor"
                class="size-6 text-neutral-700 dark:text-neutral-400"
        >
          <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M6 18 18 6M6 6l12 12"
          />
        </svg>
        </button>
      </div>
      <div class="relative w-full">
        <div class="font-extralight">Mass:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.mass}
        />
      
        <div class="font-extralight">Max Linear Velocity:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.vMax}
        />
        
        <div class="font-extralight">Angular Velocity:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.aVel}
        />
          
        <div class="font-extralight">Kinetic Friction:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.kFriction}
        />
            
        <div class="font-extralight">Drag Coefficient:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.drag}
        />
              
        <div class="font-extralight">Boundary Tolerance:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.bTolerance}
        />
                
        <div class="font-extralight">Submersible Tolerance:</div>
        <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                min="0"
                bind:value={settings.sTolerance}
        />
        </div>

      </div>
    </div>
{/if}
