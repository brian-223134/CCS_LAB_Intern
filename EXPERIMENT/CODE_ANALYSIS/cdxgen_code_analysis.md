# cdxgen

Github URL: https://github.com/cdxgen/cdxgen
비고: Parser 부분에서 Lock, pyproject 파일을 어떻게 파싱하는 지 확인
진행도: 완료

# cdxgen의 process

### 1. 설정/옵션 로드

- 현재 디렉터리에서 `.cdxgenrc`, `.cdxgen.{json,yml,yaml}`를 찾아 기본 설정으로 병합.
- yargs로 CLI 옵션 파싱(환경변수 `CDXGEN_*`도 반영). `-type` 없으면 자동 언어/플랫폼 감지로 통합 SBOM 생성.

### 2. 기본값/별칭 적용

- 타겟 경로(`args._[0]` 또는 CWD)와 `projectName` 결정.
- 실행 파일 이름에 따라 alias 처리(`obom`→os, `cbom/saasbom`→crypto/evidence 포함).
- profile/lifecycle/technique에 따라 `deep/evidence/includeCrypto/installDeps` 등을 자동 조정.

### 3. 출력 경로 준비

- `options.output`의 디렉터리가 없으면 생성. projectType 배열 중복 제거.

### 4. 보안/권한 점검 (Node 20 권한 API 고려)

- secure 모드면 fs.read/write, child-process, tmp 디렉터리 권한 등 체크 후 부족하면 경고/종료.
- root 실행, 공백 경로, 긴 경로 등에 대한 안내 메시지.

### 5. 프록시/배너/서버 모드

- HTTP(S)_PROXY 설정 시 `global-agent` 부트스트랩.
- 스폰서 배너 출력.
- `-server`면 서버를 띄우고 종료(생성 플로우로 가지 않음).

### 6. 환경 준비

- `prepareEnv(filePath, options)`로 사전 준비(캐시/임시폴더 등).

### 7. SBOM 생성

- `createBom(filePath, options)` 호출 → BOM 데이터(`bomNSData`) 생성.
- 후처리 `postProcess`로 메타데이터/annotation 추가.

### 8. 결과 저장/서명

- `options.output`에 BOM JSON 저장(pretty는 `-json-pretty`).
- ns 매핑이 있으면 `<output>.map` 저장.
- 서명 조건(`-generate-key-and-sign` 또는 서명 관련 env) 충족 시:
    - RSA 키 생성 또는 env 키 사용.
    - 각 component에 JWS 서명, 전체 BOM에도 서명 추가(+선택적 public key/JWK).

### 9. Evidence/crypto 추가 분석(옵션 시)

- `-evidence`나 `-include-crypto`면 `evinser`로 DB 준비 → 프로젝트 분석 → evinse 파일 생성.
- `-print`일 때 occurrences/reachables/services 등을 콘솔 출력.

### 10. 검증/제출/추가 포맷

- `-validate` 기본 true: 스키마 검증 실패 시 종료.
- `-server-url`+`-api-key` 있으면 Dependency-Track에 `submitBom`.
- `-export-proto`면 protobuf 바이너리(`.cdx`)도 출력.

### 11. 콘솔 출력(옵션 시)

- `-print`면 summary, formulation(옵션), dependency tree, table, crypto 테이블 등을 터미널에 렌더링.

### 12. Secure 모드 힌트

- DEBUG/TRACE 모드에서 접근한 호스트/외부 명령을 출력하며 allowlist 환경변수 설정 힌트 제공.
- 종료 로그(`thoughtEnd`) 후 프로세스 종료.

---

# 코드

- cdxgen/bin/cdxgen.js
    
    
    ```jsx
    #!/usr/bin/env node
    import { Buffer } from "node:buffer";
    import crypto from "node:crypto";
    import fs from "node:fs";
    import { basename, dirname, join, resolve } from "node:path";
    import process from "node:process";
    im
    import globalAgent from "global-agent";
    import jws from "jws";
    import { parse as _load } from "yaml";
    import yargs from "yargs";
    import { hideBin } from "yargs/helpers";
    
    import { createBom, submitBom } from "../lib/cli/index.js";
    import {
      printCallStack,
      printDependencyTree,
      printFormulation,
      printOccurrences,
      printReachables,
      printServices,
      printSponsorBanner,
      printSummary,
      printTable,
    } from "../lib/helpers/display.js";
    import { TRACE_MODE, thoughtEnd, thoughtLog } from "../lib/helpers/logger.js";
    import {
      ATOM_DB,
      commandsExecuted,
      DEBUG_MODE,
      getTmpDir,
      isMac,
      isSecureMode,
      isWin,
      remoteHostsAccessed,
      retrieveCdxgenVersion,
      safeExistsSync,
    } from "../lib/helpers/utils.js";
    import { validateBom } from "../lib/helpers/validator.js";
    import { postProcess } from "../lib/stages/postgen/postgen.js";
    import { prepareEnv } from "../lib/stages/pregen/pregen.js";
    
    // Support for config files
    const configPaths = [
      ".cdxgenrc",
      ".cdxgen.json",
      ".cdxgen.yml",
      ".cdxgen.yaml",
    ];
    let config = {};
    for (const configPattern of configPaths) {
      const configPath = join(process.cwd(), configPattern);
      if (!safeExistsSync(configPath)) {
        continue;
      }
      try {
        if (configPath.endsWith(".yml") || configPath.endsWith(".yaml")) {
          config = _load(fs.readFileSync(configPath, "utf-8"));
        } else {
          config = JSON.parse(fs.readFileSync(configPath, "utf-8"));
        }
      } catch (_e) {
        console.log("Invalid config file", configPath);
      }
    }
    
    const _yargs = yargs(hideBin(process.argv));
    
    const args = _yargs
      .env("CDXGEN")
      .parserConfiguration({
        "greedy-arrays": false,
        "short-option-groups": false,
      })
      .option("output", {
        alias: "o",
        description: "Output file. Default bom.json",
        default: "bom.json",
      })
      .option("evinse-output", {
        description:
          "Create bom with evidence as a separate file. Default bom.json",
        hidden: true,
      })
      .option("type", {
        alias: "t",
        description:
          "Project type. Please refer to https://cdxgen.github.io/cdxgen/#/PROJECT_TYPES for supported languages/platforms.",
      })
      .option("exclude-type", {
        description:
          "Project types to exclude. Please refer to https://cdxgen.github.io/cdxgen/#/PROJECT_TYPES for supported languages/platforms.",
      })
      .option("recurse", {
        alias: "r",
        type: "boolean",
        default: true,
        description:
          "Recurse mode suitable for mono-repos. Defaults to true. Pass --no-recurse to disable.",
      })
      .option("print", {
        alias: "p",
        type: "boolean",
        description: "Print the SBOM as a table with tree.",
      })
      .option("resolve-class", {
        alias: "c",
        type: "boolean",
        description: "Resolve class names for packages. jars only for now.",
      })
      .option("deep", {
        type: "boolean",
        description:
          "Perform deep searches for components. Useful while scanning C/C++ apps, live OS and oci images.",
      })
      .option("server-url", {
        description: "Dependency track url. Eg: https://deptrack.cyclonedx.io",
      })
      .option("skip-dt-tls-check", {
        type: "boolean",
        default: false,
        description: "Skip TLS certificate check when calling Dependency-Track. ",
      })
      .option("api-key", {
        description: "Dependency track api key",
      })
      .option("project-group", {
        description: "Dependency track project group",
      })
      .option("project-name", {
        description:
          "Dependency track project name. Default use the directory name",
      })
      .option("project-version", {
        description: "Dependency track project version",
        default: "",
        type: "string",
      })
      .option("project-tag", {
        description: "Dependency track project tag. Multiple values allowed.",
      })
      .option("project-id", {
        description:
          "Dependency track project id. Either provide the id or the project name and version together",
        type: "string",
      })
      .option("parent-project-id", {
        description: "Dependency track parent project id",
        type: "string",
      })
      .option("required-only", {
        type: "boolean",
        description:
          "Include only the packages with required scope on the SBOM. Would set compositions.aggregate to incomplete unless --no-auto-compositions is passed.",
      })
      .option("fail-on-error", {
        type: "boolean",
        default: isSecureMode,
        description: "Fail if any dependency extractor fails.",
      })
      .option("no-babel", {
        type: "boolean",
        description:
          "Do not use babel to perform usage analysis for JavaScript/TypeScript projects.",
      })
      .option("generate-key-and-sign", {
        type: "boolean",
        description:
          "Generate an RSA public/private key pair and then sign the generated SBOM using JSON Web Signatures.",
      })
      .option("server", {
        type: "boolean",
        description: "Run cdxgen as a server",
      })
      .option("server-host", {
        description: "Listen address",
        default: "127.0.0.1",
      })
      .option("server-port", {
        description: "Listen port",
        default: "9090",
      })
      .option("install-deps", {
        type: "boolean",
        default: !isSecureMode,
        description:
          "Install dependencies automatically for some projects. Defaults to true but disabled for containers and oci scans. Use --no-install-deps to disable this feature.",
      })
      .option("validate", {
        type: "boolean",
        default: true,
        description:
          "Validate the generated SBOM using json schema. Defaults to true. Pass --no-validate to disable.",
      })
      .option("evidence", {
        type: "boolean",
        default: false,
        description: "Generate SBOM with evidence for supported languages.",
      })
      .option("deps-slices-file", {
        description: "Path for the parsedeps slice file created by atom.",
        default: "deps.slices.json",
        hidden: true,
      })
      .option("usages-slices-file", {
        description: "Path for the usages slices file created by atom.",
        hidden: true,
      })
      .option("data-flow-slices-file", {
        description: "Path for the data-flow slices file created by atom.",
        hidden: true,
      })
      .option("reachables-slices-file", {
        description: "Path for the reachables slices file created by atom.",
        hidden: true,
      })
      .option("semantics-slices-file", {
        description: "Path for the semantics slices file.",
        default: "semantics.slices.json",
        hidden: true,
      })
      .option("openapi-spec-file", {
        description: "Path for the openapi specification file (SaaSBOM).",
        hidden: true,
      })
      .option("spec-version", {
        description: "CycloneDX Specification version to use. Defaults to 1.6",
        default: 1.6,
        type: "number",
        choices: [1.4, 1.5, 1.6, 1.7],
      })
      .option("filter", {
        description:
          "Filter components containing this word in purl or component.properties.value. Multiple values allowed.",
      })
      .option("only", {
        description:
          "Include components only containing this word in purl. Useful to generate BOM with first party components alone. Multiple values allowed.",
      })
      .option("author", {
        description:
          "The person(s) who created the BOM. Set this value if you're intending the modify the BOM and claim authorship.",
        default: "OWASP Foundation",
      })
      .option("profile", {
        description: "BOM profile to use for generation. Default generic.",
        default: "generic",
        choices: [
          "appsec",
          "research",
          "operational",
          "threat-modeling",
          "license-compliance",
          "generic",
          "machine-learning",
          "ml",
          "deep-learning",
          "ml-deep",
          "ml-tiny",
        ],
      })
      .option("lifecycle", {
        description: "Product lifecycle for the generated BOM.",
        hidden: true,
        choices: ["pre-build", "build", "post-build"],
      })
      .option("include-regex", {
        description:
          "glob pattern to include. This overrides the default pattern used during auto-detection.",
        type: "string",
      })
      .option("exclude", {
        alias: "exclude-regex",
        description: "Additional glob pattern(s) to ignore",
        type: "array",
      })
      .option("export-proto", {
        type: "boolean",
        default: false,
        description: "Serialize and export BOM as protobuf binary.",
      })
      .option("proto-bin-file", {
        description: "Path for the serialized protobuf binary.",
        default: "bom.cdx",
      })
      .option("include-formulation", {
        type: "boolean",
        default: false,
        description:
          "Generate formulation section with git metadata and build tools. Defaults to false.",
      })
      .option("include-crypto", {
        type: "boolean",
        default: false,
        description: "Include crypto libraries as components.",
      })
      .option("standard", {
        description:
          "The list of standards which may consist of regulations, industry or organizational-specific standards, maturity models, best practices, or any other requirements which can be evaluated against or attested to.",
        choices: [
          "asvs-5.0",
          "asvs-4.0.3",
          "bsimm-v13",
          "masvs-2.0.0",
          "nist_ssdf-1.1",
          "pcissc-secure-slc-1.1",
          "scvs-1.0.0",
          "ssaf-DRAFT-2023-11",
        ],
      })
      .option("no-banner", {
        type: "boolean",
        default: false,
        hidden: true,
        description:
          "Do not show the donation banner. Set this attribute if you are an active sponsor for OWASP CycloneDX.",
      })
      .option("json-pretty", {
        type: "boolean",
        default: DEBUG_MODE,
        description: "Pretty-print the generated BOM json.",
      })
      .option("feature-flags", {
        description: "Experimental feature flags to enable. Advanced users only.",
        hidden: true,
        choices: ["safe-pip-install", "suggest-build-tools", "ruby-docker-install"],
      })
      .option("min-confidence", {
        description:
          "Minimum confidence needed for the identity of a component from 0 - 1, where 1 is 100% confidence.",
        default: 0,
        type: "number",
      })
      .option("technique", {
        description: "Analysis technique to use",
        choices: [
          "auto",
          "source-code-analysis",
          "binary-analysis",
          "manifest-analysis",
          "hash-comparison",
          "instrumentation",
          "filename",
        ],
      })
      .option("tlp-classification", {
        description:
          'Traffic Light Protocol (TLP) is a classification system for identifying the potential risk associated with artefact, including whether it is subject to certain types of legal, financial, or technical threats. Refer to [https://www.first.org/tlp/](https://www.first.org/tlp/) for further information.\nThe default classification is "CLEAR"',
        choices: ["CLEAR", "GREEN", "AMBER", "AMBER_AND_STRICT", "RED"],
        default: "CLEAR",
        hidden: true,
      })
      .completion("completion", "Generate bash/zsh completion")
      .array("type")
      .array("excludeType")
      .array("filter")
      .array("only")
      .array("author")
      .array("standard")
      .array("feature-flags")
      .array("technique")
      .option("auto-compositions", {
        type: "boolean",
        default: true,
        description:
          "Automatically set compositions when the BOM was filtered. Defaults to true",
      })
      .example([
        ["$0 -t java .", "Generate a Java SBOM for the current directory"],
        [
          "$0 -t java -t js .",
          "Generate a SBOM for Java and JavaScript in the current directory",
        ],
        [
          "$0 -t java --profile ml .",
          "Generate a Java SBOM for machine learning purposes.",
        ],
        [
          "$0 -t python --profile research .",
          "Generate a Python SBOM for appsec research.",
        ],
        ["$0 --server", "Run cdxgen as a server"],
      ])
      .epilogue("for documentation, visit https://cdxgen.github.io/cdxgen")
      .config(config)
      .scriptName("cdxgen")
      .version(retrieveCdxgenVersion())
      .alias("v", "version")
      .help(false)
      .option("help", {
        alias: "h",
        type: "boolean",
        description: "Show help",
      })
      .wrap(Math.min(120, yargs().terminalWidth())).argv;
    
    if (process.env?.CDXGEN_NODE_OPTIONS) {
      process.env.NODE_OPTIONS = `${process.env.NODE_OPTIONS || ""} ${process.env.CDXGEN_NODE_OPTIONS}`;
    }
    
    if (args.help) {
      console.log(`${retrieveCdxgenVersion()}\n`);
      _yargs.showHelp();
      process.exit(0);
    }
    
    if (process.env.GLOBAL_AGENT_HTTP_PROXY || process.env.HTTP_PROXY) {
      // Support standard HTTP_PROXY variable if the user doesn't override the namespace
      if (!process.env.GLOBAL_AGENT_ENVIRONMENT_VARIABLE_NAMESPACE) {
        process.env.GLOBAL_AGENT_ENVIRONMENT_VARIABLE_NAMESPACE = "";
      }
      globalAgent.bootstrap();
      thoughtLog("Using the configured HTTP proxy. 🌐");
    }
    
    const filePath = args._[0] || process.cwd();
    if (!args.projectName) {
      if (filePath !== ".") {
        args.projectName = basename(filePath);
      } else {
        args.projectName = basename(resolve(filePath));
      }
    }
    thoughtLog(`Let's try to generate a CycloneDX BOM for the path '${filePath}'`);
    if (
      filePath.includes(" ") ||
      filePath.includes("\r") ||
      filePath.includes("\n")
    ) {
      console.log(
        `'${filePath}' contains spaces. This could lead to bugs when invoking external build tools.`,
      );
      if (isSecureMode) {
        process.exit(1);
      }
    }
    // Support for obom/cbom aliases
    if (process.argv[1].includes("obom") && !args.type) {
      args.type = "os";
      thoughtLog(
        "Ok, the user wants to generate an Operations Bill-of-Materials (OBOM).",
      );
    }
    
    /**
     * Command line options
     */
    const options = Object.assign({}, args, {
      projectType: args.type,
      multiProject: args.recurse,
      noBabel: args.noBabel || args.babel === false,
      project: args.projectId,
      deep: args.deep || args.evidence,
      output:
        isSecureMode && args.output === "bom.json"
          ? resolve(join(filePath, args.output))
          : args.output,
      exclude: args.exclude || args.excludeRegex,
      include: args.include || args.includeRegex,
    });
    // Should we create the output directory?
    const outputDirectory = dirname(options.output);
    if (
      outputDirectory &&
      outputDirectory !== process.cwd() &&
      !safeExistsSync(outputDirectory)
    ) {
      fs.mkdirSync(outputDirectory, { recursive: true });
    }
    // Filter duplicate types. Eg: -t gradle -t gradle
    if (options.projectType && Array.isArray(options.projectType)) {
      options.projectType = Array.from(new Set(options.projectType));
    }
    if (!options.projectType) {
      thoughtLog(
        "Ok, the user wants me to identify all the project types and generate a consolidated BOM document.",
      );
    }
    // Handle dedicated cbom and saasbom commands
    if (["cbom", "saasbom"].includes(process.argv[1])) {
      if (process.argv[1].includes("cbom")) {
        thoughtLog(
          "Ok, the user wants to generate Cryptographic Bill-of-Materials (CBOM).",
        );
        options.includeCrypto = true;
      } else if (process.argv[1].includes("saasbom")) {
        thoughtLog(
          "Ok, the user wants to generate a Software as a Service Bill-of-Materials (SaaSBOM). I should carefully collect the services, endpoints, and data flows.",
        );
        if (process.env?.CDXGEN_IN_CONTAINER !== "true") {
          thoughtLog(
            "Wait, I'm not running in a container. This means the chances of successfully collecting this inventory are quite low. Perhaps this is an advanced user who has set up atom and atom-tools already 🤔?",
          );
        }
      }
      options.evidence = true;
      options.specVersion = 1.6;
      options.deep = true;
    }
    if (process.argv[1].includes("cdxgen-secure")) {
      thoughtLog(
        "Ok, the user wants cdxgen to run in secure mode by default. Let's try and use the permissions api.",
      );
      console.log(
        "NOTE: Secure mode only restricts cdxgen from performing certain activities such as package installation. It does not provide security guarantees in the presence of malicious code.",
      );
      options.installDeps = false;
      process.env.CDXGEN_SECURE_MODE = true;
    }
    if (options.standard) {
      options.specVersion = 1.6;
    }
    if (options.includeFormulation) {
      thoughtLog(
        "Wait, the user wants to include formulation information. Let's warn about accidentally disclosing sensitive data via the BOM files.",
      );
      console.log(
        "NOTE: Formulation section could include sensitive data such as emails and secrets.\nPlease review the generated SBOM before distribution.\n",
      );
    }
    /**
     * Method to apply advanced options such as profile and lifecycles
     *
     * @param {object} options CLI options
     */
    const applyAdvancedOptions = (options) => {
      if (options?.profile !== "generic") {
        thoughtLog(`BOM profile to use is '${options.profile}'.`);
      } else {
        thoughtLog(
          "The user hasn't specified a profile. Should I suggest one to optimize the BOM for a specific use case or persona 🤔?",
        );
      }
      switch (options.profile) {
        case "appsec":
          options.deep = true;
          break;
        case "research":
          options.deep = true;
          options.evidence = true;
          options.includeCrypto = true;
          process.env.CDX_MAVEN_INCLUDE_TEST_SCOPE = "true";
          process.env.ASTGEN_IGNORE_DIRS = "";
          process.env.ASTGEN_IGNORE_FILE_PATTERN = "";
          break;
        case "operational":
          if (options?.projectType) {
            options.projectType.push("os");
          } else {
            options.projectType = ["os"];
          }
          break;
        case "threat-modeling":
          options.deep = true;
          options.evidence = true;
          break;
        case "license-compliance":
          process.env.FETCH_LICENSE = "true";
          break;
        case "ml-tiny":
          process.env.FETCH_LICENSE = "true";
          options.deep = false;
          options.evidence = false;
          options.includeCrypto = false;
          options.installDeps = false;
          break;
        case "machine-learning":
        case "ml":
          process.env.FETCH_LICENSE = "true";
          options.deep = true;
          options.evidence = false;
          options.includeCrypto = false;
          options.installDeps = !isSecureMode;
          break;
        case "deep-learning":
        case "ml-deep":
          process.env.FETCH_LICENSE = "true";
          options.deep = true;
          options.evidence = true;
          options.includeCrypto = true;
          options.installDeps = !isSecureMode;
          break;
        default:
          break;
      }
      if (options.lifecycle) {
        thoughtLog(
          `BOM must be generated for the lifecycle '${options.lifecycle}'.`,
        );
      }
      switch (options.lifecycle) {
        case "pre-build":
          options.installDeps = false;
          break;
        case "post-build":
          if (
            !options.projectType ||
            (Array.isArray(options.projectType) &&
              options.projectType.length > 1) ||
            ![
              "csharp",
              "dotnet",
              "container",
              "docker",
              "podman",
              "oci",
              "android",
              "apk",
              "aab",
              "go",
              "golang",
              "rust",
              "rust-lang",
              "cargo",
            ].includes(options.projectType[0])
          ) {
            console.log(
              "PREVIEW: post-build lifecycle SBOM generation is supported only for android, dotnet, go, and Rust projects. Please specify the type using the -t argument.",
            );
            process.exit(1);
          }
          options.installDeps = true;
          break;
        default:
          break;
      }
      // When the user specifies source-code-analysis as a technique, then enable deep and evidence mode.
      if (options?.technique && Array.isArray(options.technique)) {
        if (options?.technique?.includes("source-code-analysis")) {
          options.deep = true;
          options.evidence = true;
        }
        if (options.technique.length === 1) {
          thoughtLog(
            `Wait, the user wants me to use only the following technique: '${options.technique.join(", ")}'.`,
          );
        } else {
          thoughtLog(
            `Alright, I will use only the following techniques: '${options.technique.join(", ")}' for the final BOM.`,
          );
        }
      }
      if (!options.installDeps) {
        thoughtLog(
          "I must avoid any package installations and focus solely on the available artefacts, such as lock files.",
        );
      }
      return options;
    };
    applyAdvancedOptions(options);
    
    /**
     * Check for node >= 20 permissions
     *
     * @param {string} filePath File path
     * @param {Object} options CLI Options
     * @returns
     */
    const checkPermissions = (filePath, options) => {
      const fullFilePath = resolve(filePath);
      if (
        process.getuid &&
        process.getuid() === 0 &&
        process.env?.CDXGEN_IN_CONTAINER !== "true"
      ) {
        console.log(
          "\x1b[1;35mSECURE MODE: DO NOT run cdxgen with root privileges.\x1b[0m",
        );
      }
      if (!process.permission) {
        if (isSecureMode) {
          console.log(
            "\x1b[1;35mSecure mode requires permission-related arguments. These can be passed as CLI arguments directly to the node runtime or set the NODE_OPTIONS environment variable as shown below.\x1b[0m",
          );
          const childProcessArgs =
            options?.lifecycle !== "pre-build" ? " --allow-child-process" : "";
          const nodeOptionsVal = `--permission --allow-fs-read="${getTmpDir()}/*" --allow-fs-write="${getTmpDir()}/*" --allow-fs-read="${fullFilePath}/*" --allow-fs-write="${options.output}"${childProcessArgs}`;
          console.log(
            `${isWin ? "$env:" : "export "}NODE_OPTIONS='${nodeOptionsVal}'`,
          );
          if (process.env?.CDXGEN_IN_CONTAINER !== "true") {
            console.log(
              "TIP: Run cdxgen using the secure container image 'ghcr.io/cyclonedx/cdxgen-secure' for best experience.",
            );
          }
        }
        return true;
      }
      // Secure mode checks
      if (isSecureMode) {
        if (process.env?.GITHUB_TOKEN) {
          console.log(
            "Ensure that the GitHub token provided to cdxgen is restricted to read-only scopes.",
          );
        }
        if (process.permission.has("fs.read", "*")) {
          console.log(
            "\x1b[1;35mSECURE MODE: DO NOT run cdxgen with FileSystemRead permission set to wildcard.\x1b[0m",
          );
        }
        if (process.permission.has("fs.write", "*")) {
          console.log(
            "\x1b[1;35mSECURE MODE: DO NOT run cdxgen with FileSystemWrite permission set to wildcard.\x1b[0m",
          );
        }
        if (process.permission.has("worker")) {
          console.log(
            "SECURE MODE: DO NOT run cdxgen with worker thread permission! Remove `--allow-worker` argument.",
          );
        }
        if (filePath !== fullFilePath) {
          console.log(
            `\x1b[1;35mSECURE MODE: Invoke cdxgen with an absolute path to improve security. Use '${fullFilePath}' instead of '${filePath}'\x1b[0m`,
          );
          if (fullFilePath.includes(" ")) {
            console.log(
              "\x1b[1;35mSECURE MODE: Directory names containing spaces are known to cause issues. Rename the directories by replacing spaces with hyphens or underscores.\x1b[0m",
            );
          } else if (fullFilePath.length > 255 && isWin) {
            console.log(
              "Ensure 'Enable Win32 Long paths' is set to 'Enabled' by using Group Policy Editor.",
            );
          }
          return false;
        }
      }
    
      if (!process.permission.has("fs.read", filePath)) {
        console.log(
          `\x1b[1;35mSECURE MODE: FileSystemRead permission required. Please invoke cdxgen with the argument --allow-fs-read="${resolve(
            filePath,
          )}"\x1b[0m`,
        );
        return false;
      }
      if (!process.permission.has("fs.write", options.output)) {
        console.log(
          `\x1b[1;35mSECURE MODE: FileSystemWrite permission is required to create the output BOM file. Please invoke cdxgen with the argument --allow-fs-write="${options.output}"\x1b[0m`,
        );
      }
      if (options.evidence) {
        const slicesFilesKeys = [
          "deps-slices-file",
          "usages-slices-file",
          "reachables-slices-file",
        ];
        if (options?.type?.includes("swift") || options?.type?.includes("scala")) {
          slicesFilesKeys.push("semantics-slices-file");
        }
        for (const sf of slicesFilesKeys) {
          let issueFound = false;
          if (!process.permission.has("fs.write", options[sf])) {
            console.log(
              `SECURE MODE: FileSystemWrite permission is required to create the output slices file. Please invoke cdxgen with the argument --allow-fs-write="${options[sf]}"`,
            );
            if (!issueFound) {
              issueFound = true;
            }
          }
          if (issueFound) {
            return false;
          }
        }
        if (!process.permission.has("fs.write", process.env.ATOM_DB || ATOM_DB)) {
          console.log(
            `SECURE MODE: FileSystemWrite permission is required to create the output slices file. Please invoke cdxgen with the argument --allow-fs-write="${process.env.ATOM_DB || ATOM_DB}"`,
          );
          return false;
        }
        console.log(
          "TIP: Invoke cdxgen with `--allow-addons` to allow the use of sqlite3 native addon. This addon is required for evidence mode.",
        );
      } else {
        if (process.permission.has("fs.write", process.env.ATOM_DB || ATOM_DB)) {
          console.log(
            `SECURE MODE: FileSystemWrite permission is not required for the directory "${process.env.ATOM_DB || ATOM_DB}" in non-evidence mode. Consider removing the argument --allow-fs-write="${process.env.ATOM_DB || ATOM_DB}".`,
          );
          return false;
        }
      }
      if (!process.permission.has("fs.write", getTmpDir())) {
        console.log(
          `FileSystemWrite permission may be required for the TEMP directory. Please invoke cdxgen with the argument --allow-fs-write="${join(getTmpDir(), "*")}" in case of any crashes.`,
        );
        if (isMac) {
          console.log(
            "TIP: macOS doesn't use the `/tmp` prefix for TEMP directories. Use the argument shown above.",
          );
        }
      }
      if (!process.permission.has("child") && !isSecureMode) {
        console.log(
          "ChildProcess permission is missing. This is required to spawn commands for some languages. Please invoke cdxgen with the argument --allow-child-process in case of issues.",
        );
      }
      if (process.permission.has("child") && options?.lifecycle === "pre-build") {
        console.log(
          "SECURE MODE: ChildProcess permission is not required for pre-build SBOM generation. Please invoke cdxgen without the argument --allow-child-process.",
        );
        return false;
      }
      return true;
    };
    
    const needsBomSigning = ({ generateKeyAndSign }) =>
      generateKeyAndSign ||
      (process.env.SBOM_SIGN_ALGORITHM &&
        process.env.SBOM_SIGN_ALGORITHM !== "none" &&
        ((process.env.SBOM_SIGN_PRIVATE_KEY &&
          safeExistsSync(process.env.SBOM_SIGN_PRIVATE_KEY)) ||
          process.env.SBOM_SIGN_PRIVATE_KEY_BASE64));
    
    /**
     * Method to start the bom creation process
     */
    (async () => {
      // Display the sponsor banner
      printSponsorBanner(options);
      // Start SBOM server
      if (options.server) {
        const serverModule = await import("../lib/server/server.js");
        return serverModule.start(options);
      }
      // Check if cdxgen has the required permissions
      if (!checkPermissions(filePath, options)) {
        if (isSecureMode) {
          process.exit(1);
        }
        return;
      }
      prepareEnv(filePath, options);
      thoughtLog("Getting ready to generate the BOM ⚡️.");
      let bomNSData = (await createBom(filePath, options)) || {};
      if (bomNSData?.bomJson) {
        thoughtLog(
          "Tweaking the generated BOM data with useful annotations and properties.",
        );
      }
      // Add extra metadata and annotations with post processing
      bomNSData = postProcess(bomNSData, options);
      if (
        options.output &&
        (typeof options.output === "string" || options.output instanceof String)
      ) {
        const jsonFile = options.output;
        // Create bom json file
        if (bomNSData.bomJson) {
          let jsonPayload;
          if (
            typeof bomNSData.bomJson === "string" ||
            bomNSData.bomJson instanceof String
          ) {
            fs.writeFileSync(jsonFile, bomNSData.bomJson);
            jsonPayload = bomNSData.bomJson;
          } else {
            jsonPayload = JSON.stringify(
              bomNSData.bomJson,
              null,
              options.jsonPretty ? 2 : null,
            );
            fs.writeFileSync(jsonFile, jsonPayload);
            if (jsonFile.endsWith("bom.json")) {
              thoughtLog(
                `Let's save the file to "${jsonFile}". Should I suggest the '.cdx.json' file extension for better semantics?`,
              );
            } else {
              thoughtLog(`Let's save the file to "${jsonFile}".`);
            }
          }
          if (jsonPayload && needsBomSigning(options)) {
            let alg = process.env.SBOM_SIGN_ALGORITHM || "RS512";
            if (alg.includes("none")) {
              alg = "RS512";
            }
            let privateKeyToUse;
            let jwkPublicKey;
            let publicKeyFile;
            if (options.generateKeyAndSign) {
              const jdirName = dirname(jsonFile);
              publicKeyFile = join(jdirName, "public.key");
              const privateKeyFile = join(jdirName, "private.key");
              const privateKeyB64File = join(jdirName, "private.key.base64");
              const { privateKey, publicKey } = crypto.generateKeyPairSync("rsa", {
                modulusLength: 4096,
                publicKeyEncoding: {
                  type: "spki",
                  format: "pem",
                },
                privateKeyEncoding: {
                  type: "pkcs8",
                  format: "pem",
                },
              });
              fs.writeFileSync(publicKeyFile, publicKey);
              fs.writeFileSync(privateKeyFile, privateKey);
              fs.writeFileSync(
                privateKeyB64File,
                Buffer.from(privateKey, "utf8").toString("base64"),
              );
              console.log(
                "Created public/private key pairs for testing purposes",
                publicKeyFile,
                privateKeyFile,
                privateKeyB64File,
              );
              privateKeyToUse = privateKey;
              jwkPublicKey = crypto
                .createPublicKey(publicKey)
                .export({ format: "jwk" });
            } else {
              if (process.env?.SBOM_SIGN_PRIVATE_KEY) {
                privateKeyToUse = fs.readFileSync(
                  process.env.SBOM_SIGN_PRIVATE_KEY,
                  "utf8",
                );
              } else if (process.env?.SBOM_SIGN_PRIVATE_KEY_BASE64) {
                privateKeyToUse = Buffer.from(
                  process.env.SBOM_SIGN_PRIVATE_KEY_BASE64,
                  "base64",
                ).toString("utf8");
              }
              if (
                process.env.SBOM_SIGN_PUBLIC_KEY &&
                safeExistsSync(process.env.SBOM_SIGN_PUBLIC_KEY)
              ) {
                jwkPublicKey = crypto
                  .createPublicKey(
                    fs.readFileSync(process.env.SBOM_SIGN_PUBLIC_KEY, "utf8"),
                  )
                  .export({ format: "jwk" });
              } else if (process.env?.SBOM_SIGN_PUBLIC_KEY_BASE64) {
                jwkPublicKey = Buffer.from(
                  process.env.SBOM_SIGN_PUBLIC_KEY_BASE64,
                  "base64",
                ).toString("utf8");
              }
            }
            try {
              // Sign the individual components
              // Let's leave the services unsigned for now since it might require additional cleansing
              const bomJsonUnsignedObj = JSON.parse(jsonPayload);
              for (const comp of bomJsonUnsignedObj.components) {
                const compSignature = jws.sign({
                  header: { alg },
                  payload: comp,
                  privateKey: privateKeyToUse,
                });
                const compSignatureBlock = {
                  algorithm: alg,
                  value: compSignature,
                };
                if (jwkPublicKey) {
                  compSignatureBlock.publicKey = jwkPublicKey;
                }
                comp.signature = compSignatureBlock;
              }
              const signature = jws.sign({
                header: { alg },
                payload: JSON.stringify(bomJsonUnsignedObj, null, 2),
                privateKey: privateKeyToUse,
              });
              if (signature) {
                const signatureBlock = {
                  algorithm: alg,
                  value: signature,
                };
                if (jwkPublicKey) {
                  signatureBlock.publicKey = jwkPublicKey;
                }
                bomJsonUnsignedObj.signature = signatureBlock;
                fs.writeFileSync(
                  jsonFile,
                  JSON.stringify(
                    bomJsonUnsignedObj,
                    null,
                    options.jsonPretty ? 2 : null,
                  ),
                );
                thoughtLog(`Signing the BOM file "${jsonFile}".`);
                if (publicKeyFile) {
                  // Verifying this signature
                  const signatureVerification = jws.verify(
                    signature,
                    alg,
                    fs.readFileSync(publicKeyFile, "utf8"),
                  );
                  if (signatureVerification) {
                    console.log(
                      "SBOM signature is verifiable with the public key and the algorithm",
                      publicKeyFile,
                      alg,
                    );
                  } else {
                    console.log("SBOM signature verification was unsuccessful");
                    console.log(
                      "Check if the public key was exported in PEM format",
                    );
                  }
                }
              }
            } catch (ex) {
              console.log("SBOM signing was unsuccessful", ex);
              console.log("Check if the private key was exported in PEM format");
            }
          }
        }
        // bom ns mapping
        if (bomNSData.nsMapping && Object.keys(bomNSData.nsMapping).length) {
          const nsFile = `${jsonFile}.map`;
          fs.writeFileSync(nsFile, JSON.stringify(bomNSData.nsMapping));
        }
      } else if (!options.print) {
        if (bomNSData.bomJson) {
          console.log(
            JSON.stringify(bomNSData.bomJson, null, options.jsonPretty ? 2 : null),
          );
        } else {
          console.log("Unable to produce BOM for", filePath);
          console.log("Try running the command with -t <type> or -r argument");
        }
      }
      // Evidence generation
      if (options.evidence || options.includeCrypto) {
        // Set the evinse output file to be the same as output file
        if (!options.evinseOutput) {
          options.evinseOutput = options.output;
        }
        const evinserModule = await import("../lib/evinser/evinser.js");
        options.projectType = options.projectType || ["java"];
        const evinseOptions = {
          _: args._,
          input: options.output,
          output: options.evinseOutput,
          language: options.projectType,
          dbPath: process.env.ATOM_DB || ATOM_DB,
          skipMavenCollector: false,
          force: false,
          withReachables: options.deep,
          usagesSlicesFile: options.usagesSlicesFile,
          dataFlowSlicesFile: options.dataFlowSlicesFile,
          reachablesSlicesFile: options.reachablesSlicesFile,
          semanticsSlicesFile: options.semanticsSlicesFile,
          openapiSpecFile: options.openapiSpecFile,
          includeCrypto: options.includeCrypto,
          specVersion: options.specVersion,
          profile: options.profile,
          jsonPretty: options.jsonPretty,
        };
        const dbObjMap = await evinserModule.prepareDB(evinseOptions);
        if (dbObjMap) {
          const sliceArtefacts = await evinserModule.analyzeProject(
            dbObjMap,
            evinseOptions,
          );
          const evinseJson = evinserModule.createEvinseFile(
            sliceArtefacts,
            evinseOptions,
          );
          bomNSData.bomJson = evinseJson;
          if (options.print && evinseJson) {
            printOccurrences(evinseJson);
            printCallStack(evinseJson);
            printReachables(sliceArtefacts);
            printServices(evinseJson);
          }
        }
      }
      // Perform automatic validation
      if (options.validate && bomNSData?.bomJson) {
        thoughtLog("Wait, let's check the generated BOM file for any issues.");
        if (!validateBom(bomNSData.bomJson)) {
          process.exit(1);
        }
        thoughtLog("✅ BOM file looks valid.");
      }
      thoughtEnd();
      // Automatically submit the bom data
      // biome-ignore lint/suspicious/noDoubleEquals: yargs passes true for empty values
      if (options.serverUrl && options.serverUrl != true && options.apiKey) {
        try {
          await submitBom(options, bomNSData.bomJson);
        } catch (err) {
          console.log(err);
          process.exit(1);
        }
      }
      // Protobuf serialization
      if (options.exportProto) {
        const protobomModule = await import("../lib/helpers/protobom.js");
        protobomModule.writeBinary(bomNSData.bomJson, options.protoBinFile);
        thoughtLog("BOM file is also available in .proto format!");
      }
      if (options.print && bomNSData.bomJson && bomNSData.bomJson.components) {
        printSummary(bomNSData.bomJson);
        if (options.includeFormulation) {
          printFormulation(bomNSData.bomJson);
        }
        printDependencyTree(bomNSData.bomJson);
        printTable(bomNSData.bomJson);
        // CBOM related print
        if (options.includeCrypto) {
          printTable(bomNSData.bomJson, ["cryptographic-asset"]);
          printDependencyTree(bomNSData.bomJson, "provides");
        }
      }
      if (
        (DEBUG_MODE || TRACE_MODE) &&
        (!process.env?.CDXGEN_ALLOWED_HOSTS ||
          !process.env?.CDXGEN_ALLOWED_COMMANDS)
      ) {
        let allowListSuggestion = "";
        const envPrefix = isWin ? "set $env:" : "export ";
        if (remoteHostsAccessed.size) {
          allowListSuggestion = `${envPrefix}CDXGEN_ALLOWED_HOSTS="${Array.from(remoteHostsAccessed).join(",")}"\n`;
        }
        if (commandsExecuted.size) {
          allowListSuggestion = `${allowListSuggestion}${envPrefix}CDXGEN_ALLOWED_COMMANDS="${Array.from(commandsExecuted).join(",")}"\n`;
        }
        if (allowListSuggestion) {
          console.log(
            "SECURE MODE: cdxgen supports allowlists for remote hosts and external commands. Set the following environment variables to get started.",
          );
          console.log(allowListSuggestion);
        }
      }
    })();
    
    ```
    
- cdxgen/lib/cli/index.js - createBom
    
    ```jsx
    export async function createBom(path, options) {
      let { projectType } = options;
      if (!projectType) {
        projectType = [];
      }
      let exportData;
      let isContainerMode = false;
      // Docker and image archive support
      if (path.endsWith(".tar") || path.endsWith(".tar.gz")) {
        exportData = await exportArchive(path, options);
        ...
        isContainerMode = true;
      } else if (
        (options.projectType &&
          !options.projectType?.includes("universal") &&
          hasAnyProjectType(["oci"], options, false)) ||
        path.startsWith("docker.io") ||
        path.startsWith("quay.io") ||
        path.startsWith("ghcr.io") ||
        path.startsWith("mcr.microsoft.com") ||
        path.includes("@sha256") ||
        path.includes(":latest")
      ) {
        exportData = await exportImage(path, options);
        ...
        isContainerMode = true;
      } else if (
        !options.projectType?.includes("universal") &&
        hasAnyProjectType(["oci-dir"], options, false)
      ) {
        isContainerMode = true;
        exportData = { ... };
        exportData.pkgPathList = getPkgPathList(exportData, undefined);
      }
      if (isContainerMode) {
        options.multiProject = true;
        options.installDeps = false;
        options.projectType = ["oci"];   // 강제로 oci로 세팅
        options.path = path;
        options.parentComponent = {};
        const inspectData = exportData?.inspectData;
        if (inspectData?.RepoDigests && inspectData.RepoTags ...) {
          const repoTag = inspectData.RepoTags[0];
          ...
          options.parentComponent = {
            name: tmpA[0],
            version: tmpA[1],
            type: "container",
            purl: `pkg:oci/${inspectData.RepoDigests[0]}`,
            _integrity: inspectData.RepoDigests[0].replace("sha256:", "sha256-"),
            "bom-ref": decodeURIComponent(purl),
          };
        }
    ```
    
- cdxgen/lib/helpers/utils.js - hasAnyProjectType
    
    ```jsx
    for (const abt of Object.keys(PROJECT_TYPE_ALIASES)) {
        if (
          PROJECT_TYPE_ALIASES[abt].filter((pt) =>
            new Set(options?.projectType).has(pt),
          ).length
        ) {
          baseProjectTypes.push(abt);
        }
        if (
          PROJECT_TYPE_ALIASES[abt].filter((pt) => new Set(projectTypes).has(pt))
            .length
        ) {
          allProjectTypes.push(abt);
        }
        if (
          PROJECT_TYPE_ALIASES[abt].filter((pt) =>
            new Set(options?.excludeType).has(pt),
          ).length
        ) {
          baseExcludeTypes.push(abt);
        }
      }
    ```
    
    - baseProjectTypes: 사용자가 t ...로 지정한 타입(options.projectType)이 어떤 별칭이든, 대응되는 대표 타입(abt)을 추가
    - allProjectTypes: 함수 인자로 들어온 projectTypes(현재 검사 대상)도 동일하게 대표 타입(abt)을 추가
    - baseExcludeTypes: 사용자가 -exclude-type ...로 지정한 타입(options.excludeType)을 대표 타입으로 변환해서 추가

---

## 실행 과정

- **실행 요약**
    - CLI 엔트리포인트(옵션 구성/준비)
        - (IIFE) main: cdxgen.js:803
        - applyAdvancedOptions(options) (profile/lifecycle에 따라 옵션 보정): cdxgen.js:275
        - checkPermissions(filePath, options) (secure mode/permission 점검): cdxgen.js:410
        - prepareEnv(filePath, options) (사전 환경 준비): cdxgen.js:827
    - BOM 생성 진입(라우팅)
        - createBom(filePath, options) : index.js:8430
            - projectType 분기(여기서는 `python`) → createPythonBom(path, options) 호출: index.js:8581-8589
    - Python SBOM 생성(이번 케이스: pyproject + poetry.lock 중심)
        - createPythonBom(path, options) : index.js:3721
            - createDefaultParentComponent(path, "pypi", options) (부모 기본값 생성): index.js:246
            - parsePyProjectTomlFile(pyProjectFile) (부모 컴포넌트/직접의존/그룹/워크스페이스 읽기): utils.js:5643 부근(함수 정의 위치는 파일 내 검색 필요)
            - (poetry/pdm/uv lock 발견 시) 각 lock에 대해:
                - parsePyLockData(lockData, lockFile) (락파일→pkgList + dependenciesList + rootList): utils.js:5816
                - trimComponents(pkgList) (중복/정규화): utils.js 내 유틸(정의 위치는 검색 필요)
                - mergeDependencies(dependencies, retMap.dependenciesList, parentComponent) (여러 소스에서 나온 dependency 합치기): utils.js 내 유틸(정의 위치는 검색 필요)
                - (부모→직접의존 연결 보완) dependencies.splice(0, 0, { ref: parentComponent["bom-ref"], dependsOn: [...] }) : index.js:3902-3914
            - 최종 조립 호출:
                - buildBomNSData(options, pkgList, "pypi", { dependencies, parentComponent, ... }): index.js:4203
                    - 내부에서:
                        - determineParentComponent(options) (옵션 기반 부모 확정): index.js:1377-1386 주변
                        - addMetadata(parentComponent, options, context) → metadata 구성: index.js:1387-1389
                        - listComponents(options, allImports, pkgInfo, ptype) → components[] 구성: index.js:1389-1390
    - 후처리(메타데이터/필터/주석 추가) + 저장
        - postProcess(bomNSData, options) : postgen.js:52
            - filterBom(...) → 필터링: postgen.js:68 근처
            - applyStandards(...) : postgen.js:183
            - applyMetadata(...) (예: `cdx:bom:componentSrcFiles` 자동 생성): postgen.js:74
            - (spec>=1.6) annotate(...) : postgen.js:69-72
        - fs.writeFileSync(options.output, JSON.stringify(bomNSData.bomJson, ...)) 로 capstone.json 저장: cdxgen.js:856-878
        - (옵션 기본 true) validateBom(bomNSData.bomJson) 스키마 검증: cdxgen.js:1068-1077
- ParsePyLock
    
    parsePyLockData에서 “의존성 해석”은 크게 2단계로 진행됩니다.
    
    1. 락파일의 각 패키지를 읽으면서 “간선(부모→자식)”을 수집 (depsMap 채우기)
    2. depsMap에 모인 값들을 “CycloneDX dependencies 섹션” 형태로 정규화/확정 (dependenciesList 만들기)
    
    아래는 그 큰 흐름을 코드 기준으로 설명한 것입니다.
    
    ---
    
    **1) 의존성 간선 수집: depsMap(부모 bom-ref → 자식 ref set)**
    
    - 각 패키지 apkg를 읽어 pkg를 만들고, 해당 패키지의 key(pkg["bom-ref"])에 대해 빈 Set을 준비합니다:
        - depsMap 초기화: utils.js:6048-6053
    - 이후 apkg.dependencies + dev-dependencies + optional-dependencies를 합쳐서 “이 패키지가 무엇에 의존하는지”를 depsMap에 추가합니다.
        - dev/optional를 펼쳐서 배열로 합침: utils.js:6066-6096
    - 포맷별 분기(중요)
        - PDM 계열처럼 apkg.dependencies가 “배열”인 경우
            - 의존 문자열에서 패키지명만 뽑아냅니다(예: `"msgpack>=0.5.2"` → `"msgpack"`).
            - 뽑은 이름을 depsMap에 추가할 때, 이미 같은 이름의 패키지가 앞에서 만들어져 있으면(existingPkgMap[nameStr]) “그 패키지의 bom-ref(pkg:pypi/..@..)”를 넣고, 아니면 일단 이름 문자열을 넣습니다.
            - 이 구간이 “실제 의존성 해석(문자열→패키지명 추출 + 참조 연결)”의 핵심입니다:
                - utils.js:6104-6117
        - Poetry/uv 계열처럼 apkg.dependencies가 “객체(map)”인 경우
            - 키(의존 패키지 이름)만 꺼내서 동일하게 depsMap에 추가합니다:
                - utils.js:6175-6180
    - (워크스페이스 관련) “workspace가 직접 의존한 패키지”라는 관계도 별도의 간선으로 넣습니다.
        - 예: 어떤 workspace 모듈이 특정 패키지를 직접 의존하면 workspaceRef -> 패키지 bom-ref 간선을 depsMap에 추가:
            - utils.js:6054-6065
    
    요약하면, 이 단계는 “락파일에 적힌 dependency 표현(배열/객체)을 읽어서 → 부모 패키지 bom-ref 기준으로 자식 목록을 Set에 축적”하는 단계입니다.
    
    ---
    
    **2) 의존성 참조 정규화: depsMap → dependenciesList**
    
    depsMap은 중간 단계라서 값이 섞여 있을 수 있습니다(이미 pkg:인 것도 있고, 그냥 `"idna"` 같은 문자열도 있음). 그래서 다음 단계에서 “진짜 ref”로 정규화합니다.
    
    - depsMap의 각 부모 key에 대해, 자식들을 순회하면서 depRef를 결정합니다:
        - utils.js:6183-6204
    - depRef 결정 규칙(우선순위)
        1. 이미 pkg:로 시작하면 그대로 사용
        2. 아니면 existingPkgMap[adep]로 이름→bom-ref 매핑
        3. 아니면 py${adep} 형태도 시도(이름이 `pyfoo`로 들어간 케이스 보정)
        4. 아니면 /_ 치환 보정(예: my-pkg vs `my_pkg`)
        - 이게 “같은 패키지를 가리키는 표현 차이”를 흡수하는 핵심 보정 로직입니다: utils.js:6188-6203
    - 이렇게 확정된 depRef들만 모아서 CycloneDX 형태로 push:
        - utils.js:6249-6254
    
    결과적으로 dependenciesList는 capstone.json의 "dependencies": [{ ref, dependsOn }]와 같은 구조의 직접 원천이 됩니다.
    

- poetry.lock 파일의 파싱 예시
    
    ## 1) 0단계: 초기화(맨 위 변수들)
    
    함수 시작 직후 아래가 비어있는 상태로 만들어집니다.
    
    - `pkgList = []` : SBOM `components`에 넣을 패키지 컴포넌트들
    - `dependenciesList = []` : SBOM `dependencies`(ref→dependsOn) 결과
    - `depsMap = {}` : “부모 bom-ref → 자식들 Set” 임시 그래프
    - `existingPkgMap = {}` : “패키지 이름(lowercase) → bom-ref” 매핑 (나중에 이름을 bom-ref로 바꾸는 용도)
    - `pkgBomRefMap = {}` : “bom-ref → pkg 객체” 매핑
    - `rootList = []` : “직접 의존성”으로 판단된 패키지들 (pyproject.toml이 있어야 채워짐)
    
    이번 업로드에는 `pyproject.toml`이 없어서(또는 같은 디렉토리에 없다면),
    
    - `directDepsKeys = {}` / `groupDepsKeys = {}` 그대로
    - `parentComponent`도 없음
    - 워크스페이스 로직도 실행되지 않음
    - 따라서 **`rootList`는 끝까지 비어 있을 가능성이 큽니다.**
    
    ---
    
    ## 2) 1단계: `poetry.lock`를 TOML로 파싱
    
    ```jsx
    lockTomlObj = toml.parse(lockData)
    ```
    
    `poetry.lock`는 TOML이라 여기서 성공하고,
    
    - `lockTomlObj.package` = `[[package]]` 배열(51개)
    - `lockTomlObj.metadata` = `lock-version`, `python-versions`, `content-hash` 등
    
    이제부터 **`lockTomlObj.package`에 나온 순서대로** for-loop를 돕니다.
    
    ---
    
    ## 3) 2단계: `[[package]]`를 한 개씩 돌며 “컴포넌트 생성 + depsMap 채우기”
    
    ### 예시 A) 2번째 패키지 `anyio 4.12.0`이 처리되는 순간(파일에서 index=1)
    
    `poetry.lock` 앞부분은 이런 순서예요(처음 10개):
    
    1. annotated-types 0.7.0
    2. anyio 4.12.0
    3. asgiref 3.9.1
    4. boto3 1.40.33
    5. botocore 1.40.33
    6. certifi 2025.8.3
        
        …
        
    
    ### (A-1) 컴포넌트(pkg) 기본 생성
    
    `anyio`에 대해 이런 객체를 만듭니다(핵심만):
    
    - `name="anyio"`, `version="4.12.0"`, `description=...`
    - `python-versions`가 있으므로 속성 추가:
        - `properties += { name:"cdx:pypi:requiresPython", value:">=3.9" }`
    - PURL / bom-ref 생성:
        - `purl = "pkg:pypi/anyio@4.12.0"`
        - `bom-ref = "pkg:pypi/anyio@4.12.0"`
    - evidence 추가(“lockFile 기반 식별”):
        - `evidence.identity.field="purl"`
        - `methods[0].technique="manifest-analysis"`
        - `methods[0].value=lockFile`
    
    ### (A-2) 이름→bom-ref 인덱스 등록
    
    이 시점에 맵들이 이렇게 갱신됩니다.
    
    - `existingPkgMap["anyio"] = "pkg:pypi/anyio@4.12.0"`
    - `pkgBomRefMap["pkg:pypi/anyio@4.12.0"] = (anyio pkg 객체)`
    - `pkgList.push(anyio pkg)`
    - `depsMap["pkg:pypi/anyio@4.12.0"] = new Set()`
    
    ### (A-3) 의존성 읽어서 depsMap에 “일단” 넣기
    
    `poetry.lock`에서 anyio의 dependencies는:
    
    - `idna`
    - `typing_extensions` (주의: underscore)
    
    그런데 **anyio가 등장하는 시점(index=1)**에는
    
    - `idna` 패키지가 아직 루프에서 안 나왔고(index=22)
    - `typing-extensions`도 아직 안 나왔어요(index=46)
    
    그래서 기존에 `existingPkgMap`에 없어서, 아래처럼 **문자열 그대로** 들어갑니다:
    
    - `depsMap["pkg:pypi/anyio@4.12.0"] = {"idna", "typing_extensions"}`
    
    즉, **1차 루프에서는 “bom-ref로 못 바꾸면 이름 문자열로 임시 저장”**해요.
    
    ---
    
    ### 예시 B) 22번째 패키지 `httpx 0.28.1` 처리(index=21)
    
    `httpx`의 dependencies는:
    
    - anyio
    - certifi
    - httpcore
    - idna
    
    그런데 httpx가 처리되는 시점(index=21)에는
    
    - anyio(index=1) ✅ 이미 등록됨 → bom-ref로 즉시 넣을 수 있음
    - certifi(index=5) ✅ 등록됨
    - httpcore(index=20) ✅ 등록됨
    - idna(index=22) ❌ 아직 등록 전
    
    그래서 httpx 처리 직후 depsMap은 이렇게 됩니다:
    
    - `depsMap["pkg:pypi/httpx@0.28.1"] = {`
        - `"pkg:pypi/anyio@4.12.0"`, ✅
        - `"pkg:pypi/certifi@2025.8.3"`, ✅
        - `"pkg:pypi/httpcore@1.0.9"`, ✅
        - `"idna"` ❌(문자열 임시)
        - `}`
    
    ---
    
    ### 예시 C) “Poetry.lock 특유의 케이스”가 왜 중요한지: `dj-rest-auth 7.0.1` 처리(index=11)
    
    `dj-rest-auth` dependencies는:
    
    - `Django` (대문자 D)
    - `djangorestframework`
    
    `dj-rest-auth`가 처리될 때는 django가 아직 등장 전(index=12)이기도 하지만, 더 중요한 건 **키가 `Django`(대문자)** 라는 점입니다.
    
    이 함수는 `existingPkgMap` 키를 `name.toLowerCase()`로 저장해두고,
    
    나중에 찾을 때도 “대소문자/정규화”를 충분히 해주지 않아서,
    
    - depsMap에 `"Django"`가 문자열로 들어간 뒤
    - 2차 패스에서도 `"Django"`를 `"django"`로 바꿔 찾아보지 못하면
        
        **의존성 엣지가 누락**될 수 있어요.
        
    
    (이건 아래 4단계에서 실제로 “누락되는 방식”을 보여줄게요.)
    
    ---
    
    ## 4) 3단계: 2차 패스( depsMap → dependenciesList 확정 )
    
    루프가 끝나면, 이제 **모든 패키지가 existingPkgMap에 등록된 상태**예요.
    
    이때 아래를 실행합니다:
    
    ```jsx
    for (const keyofObject.keys(depsMap)) {
    for (const adepof depsMap[key]) {
    // adep가 문자열이면 existingPkgMap에서 bom-ref로 해석 시도
      }
      dependenciesList.push({ref:key,dependsOn:[...] })
    }
    
    ```
    
    ### 예시 B 결과: httpx는 idna까지 bom-ref로 확정됨
    
    httpx의 depsMap에는 `"idna"`가 문자열로 있었는데, 이제는
    
    - `existingPkgMap["idna"] = "pkg:pypi/idna@3.10"` 이 존재하므로
        
        2차 패스에서 해결됩니다.
        
    
    최종적으로 `dependenciesList`에는 대략 이런 항목이 들어가요:
    
    - `ref: "pkg:pypi/httpx@0.28.1"`
    - `dependsOn: [`
        - `"pkg:pypi/anyio@4.12.0"`,
        - `"pkg:pypi/certifi@2025.8.3"`,
        - `"pkg:pypi/httpcore@1.0.9"`,
        - `"pkg:pypi/idna@3.10"`
        - `]`
    
    ### 예시 A 결과: anyio는 **idna는 연결되지만 typing_extensions는 누락될 수 있음**
    
    anyio depsMap은 `{ "idna", "typing_extensions" }` 였죠.
    
    - `"idna"` → `pkg:pypi/idna@3.10` 으로 잘 해석됨 ✅
    - `"typing_extensions"` → 문제 ❌
    
    왜냐하면 poetry.lock 안의 실제 패키지 이름은 `typing-extensions`(하이픈)이고,
    
    `existingPkgMap`에는 `"typing-extensions"`(소문자) 키만 있어요.
    
    이 함수의 해석 로직은 `- → _`만 시도하고(`adep.replace(/-/g,"_")`),
    
    **`_ → -`는 시도하지 않아서**,
    
    `typing_extensions` → `typing-extensions`로 매칭이 안 되면 엣지가 빠집니다.
    
    그래서 결과가 이렇게 될 수 있어요:
    
    - `ref: "pkg:pypi/anyio@4.12.0"`
    - `dependsOn: ["pkg:pypi/idna@3.10"]` ✅
    - (원래 기대: typing-extensions도 있어야 함)
    
    ### 예시 C 결과: `Django`(대문자)도 누락될 수 있음
    
    `dj-rest-auth`의 depsMap에 `"Django"`가 들어갔는데,
    
    2차 패스에서 `existingPkgMap["Django"]`는 없고(키는 소문자),
    
    대소문자 정규화를 충분히 안 하면 **django 엣지가 누락**됩니다.
    
    그래서 실제로는:
    
    - `dj-rest-auth` → `djangorestframework`는 연결되는데
    - `dj-rest-auth` → `django`는 빠질 수 있어요.
    
    ---
    
    ## 5) 4단계: getPyMetadata로 pkgList 메타데이터 보강
    
    마지막으로:
    
    ```jsx
    pkgList =awaitgetPyMetadata(pkgList,false);
    
    ```
    
    을 호출해서 PyPI 메타데이터(홈페이지/라이선스/추가 설명 등)를 더 붙일 수 있습니다(설치는 구현에 따라).
    
    ---
    
    ## 6) Poetry.lock에서 “이 함수가 그대로는 활용 못 하는 정보”도 같이 알아두기
    
    네 `poetry.lock`의 각 `[[package]]`에는 이런 정보가 있어요:
    
    ```toml
    files = [
      {file="anyio-4.12.0-py3-none-any.whl", hash="sha256:..."},
      {file="anyio-4.12.0.tar.gz", hash="sha256:..."},
    ]
    groups = ["main"]
    
    ```
    
    그런데 이 함수는 **파일 아티팩트 처리**를
    
    `lockTomlObj.metadata.files[pkg.name]` 쪽에서만 하도록 되어 있어서,
    
    `poetry.lock`의 `package.files`는 그대로면 **SBOM의 file 컴포넌트로 내려가지 않습니다.**
    
    즉, poetry.lock 기준으로 “휠/소스 tar.gz 해시까지 SBOM에 넣고 싶다”면
    
    `apkg.files`를 읽는 분기 추가가 필요해요.
    
    ---
    
    ## 한 번에 정리: “생성 순서”만 초압축
    
    1. lockData 읽기 → TOML 파싱
    2. `[[package]]` 순서대로:
        - pkg 객체 생성(purl/bom-ref/evidence/properties)
        - `existingPkgMap`, `pkgBomRefMap`, `pkgList`, `depsMap[pkg]` 갱신
        - deps는 “가능하면 bom-ref, 아니면 문자열”로 depsMap에 임시 저장
    3. 모든 패키지 처리 후 2차 패스:
        - depsMap의 문자열 deps를 existingPkgMap으로 bom-ref로 해석
        - `dependenciesList` 확정
    4. getPyMetadata로 pkgList 보강 후 return
- poetry.lock에 있는 dj-rest-auth 패키지의 처리 방식
    
    ## 0) poetry.lock의 dj-rest-auth 구간(원본 형태)
    
    `poetry.lock`에는 이렇게 들어있어요:
    
    - `name = "dj-rest-auth"`
    - `version = "7.0.1"`
    - `python-versions = ">=3.8"`
    - `[package.dependencies]`
        - `Django = ">=4.2,<6.0"`
        - `djangorestframework = ">=3.13.0"`
    - `[package.extras]`
        - `with-social = ["django-allauth[socialaccount] (>=64.0.0)"]`
    
    (중요 포인트: **Django가 대문자**로 적혀 있음)
    
    ---
    
    ## 1) 1차 루프에서 dj-rest-auth “컴포넌트”가 만들어지는 과정
    
    `for (const apkg of lockTomlObj.package || [])`에서 dj-rest-auth를 만나면, 다음 순서로 처리돼요.
    
    ### 1-1) pkg 객체 생성
    
    `pkg`는 대략 이렇게 만들어집니다(핵심 필드만):
    
    ```json
    {
    "name":"dj-rest-auth",
    "version":"7.0.1",
    "description":"Authentication and Registration in Django Rest Framework",
    "properties":[
    {"name":"cdx:pypi:requiresPython","value":">=3.8"}
    ],
    "purl":"pkg:pypi/dj-rest-auth@7.0.1",
    "bom-ref":"pkg:pypi/dj-rest-auth@7.0.1",
    "evidence":{
    "identity":{
    "field":"purl",
    "confidence":1,
    "methods":[
    {"technique":"manifest-analysis","confidence":1,"value":"<lockFile 경로>"}
    ]
    }
    }
    }
    
    ```
    
    - `python-versions`가 있으니 `cdx:pypi:requiresPython` 속성이 붙어요.
    - `purl`/`bom-ref`는 **항상 `pkg:pypi/<name>@<version>`** 형태로 생성합니다.
    - 지금은 `pyproject.toml`을 못 읽는 상황이라(또는 미지정/미탐색) `SrcFile`, `rootList(직접의존성)`, 그룹정보 등은 거의 비어 있게 끝나는 편입니다.
    
    ### 1-2) 인덱스 맵 등록(이게 2차 루프의 핵심 기반)
    
    이 시점에 아래가 갱신됩니다:
    
    - `existingPkgMap["dj-rest-auth"] = "pkg:pypi/dj-rest-auth@7.0.1"`
        - **주의:** 저장할 때 `toLowerCase()`라 키는 항상 소문자
    - `pkgBomRefMap["pkg:pypi/dj-rest-auth@7.0.1"] = (dj-rest-auth pkg 객체)`
    - `pkgList.push(dj-rest-auth pkg)`
    - `depsMap["pkg:pypi/dj-rest-auth@7.0.1"] = new Set()`
    
    ---
    
    ## 2) 1차 루프에서 “의존성 엣지(depsMap)”가 쌓이는 과정
    
    `poetry.lock`의 `[package.dependencies]`는 TOML 테이블이라 JS 객체로 파싱되고,
    
    코드는 이 분기로 들어갑니다:
    
    ```jsx
    }elseif (apkg.dependencies &&Object.keys(apkg.dependencies).length) {
    for (const apkgDepofObject.keys(apkg.dependencies)) {
        depsMap[pkg["bom-ref"]].add(existingPkgMap[apkgDep] || apkgDep);
      }
    }
    ```
    
    dj-rest-auth의 `Object.keys(apkg.dependencies)`는:
    
    - `"Django"` (대문자!)
    - `"djangorestframework"`
    
    그런데 **dj-rest-auth가 처리되는 시점에는** `django`, `djangorestframework` 패키지가 아직 루프에서 처리되기 전이라 `existingPkgMap[...]`에 없습니다.
    
    그래서 depsMap에는 “bom-ref”가 아니라 **문자열 그대로** 들어갑니다:
    
    ```jsx
    depsMap["pkg:pypi/dj-rest-auth@7.0.1"] =Set {
    "Django",
    "djangorestframework"
    }
    ```
    
    > 여기까지가 1차 루프(패키지 단위 생성 + 엣지 임시 수집)에서 dj-rest-auth가 만드는 결과이다.
    > 
    
    ---
    
    ## 3) 1차 루프가 계속 돌면서(중요): django / djangorestframework가 뒤에 등록됨
    
    dj-rest-auth 다음에 곧바로 `django 5.2.6`이 나오고, 그때:
    
    - `existingPkgMap["django"] = "pkg:pypi/django@5.2.6"`
    
    그리고 조금 뒤에 `djangorestframework 3.16.1`이 나오면:
    
    - `existingPkgMap["djangorestframework"] = "pkg:pypi/djangorestframework@3.16.1"`
    
    즉, 2차 루프 시점에는 “이름→bom-ref” 매핑이 다 준비된 상태가 됩니다.
    
    ---
    
    ## 4) 2차 루프에서 dependenciesList 확정될 때 dj-rest-auth는 어떻게 되나?
    
    2차 루프는 `depsMap`을 돌면서 각 `adep`를 **bom-ref로 해석**하려고 합니다:
    
    ```jsx
    if (adep.startsWith("pkg:")) depRef = adep;
    elseif (existingPkgMap[adep]) depRef = existingPkgMap[adep];
    elseif (existingPkgMap[`py${adep}`]) ...
    elseif (existingPkgMap[adep.replace(/-/g,"_")]) ...
    ```
    
    ### 4-1) djangorestframework는 성공 ✅
    
    - `adep = "djangorestframework"`
    - `existingPkgMap["djangorestframework"]`가 있으므로
        - `depRef = "pkg:pypi/djangorestframework@3.16.1"` ✅
    - 최종 dependsOn에 포함됨
    
    ### 4-2) Django는 실패 ❌ (이게 핵심)
    
    - `adep = "Django"`
    - 하지만 `existingPkgMap`의 키는 **소문자**로만 저장되어 있어서
        - `existingPkgMap["Django"]`는 없음 ❌
    - `"pyDjango"` 같은 키도 없음
    - `→ _` 변환도 Django에는 의미 없음
    - 결론: `depRef`가 만들어지지 않아서 **dependsOn에서 빠집니다.**
    
    ### 4-3) 그래서 최종 dependenciesList의 dj-rest-auth 항목은 이렇게 확정될 가능성이 큼
    
    ```json
    {
    "ref":"pkg:pypi/dj-rest-auth@7.0.1",
    "dependsOn":[
    "pkg:pypi/djangorestframework@3.16.1"
    ]
    }
    ```
    
    즉, **dj-rest-auth → django 직접 엣지가 누락**됩니다.
    
    ---
    
    ## 5) “그래도 django가 완전히 사라지나?” (전이 의존성 관점)
    
    `djangorestframework` 쪽 엔트리를 보면 `[package.dependencies] django = ">=4.2"` 처럼 **소문자**라서,
    
    이건 2차 루프에서 정상적으로 매핑됩니다.
    
    그래서 SBOM 그래프는 보통 이렇게 됩니다:
    
    - `dj-rest-auth@7.0.1` → `djangorestframework@3.16.1` ✅
    - `djangorestframework@3.16.1` → `django@5.2.6` ✅
    - 하지만 `dj-rest-auth@7.0.1` → `django@5.2.6` ❌ (직접 엣지 누락)
    
    전이로는 django가 보이지만, “직접 의존성”을 중시하는 분석에서는 차이가 생긴다.
    
    ---
    
    ## 6) dj-rest-auth의 extras(with-social)가 왜 SBOM에 안 잡히나?
    
    `poetry.lock`에는:
    
    - `[package.extras] with-social = ["django-allauth[socialaccount] (>=64.0.0)"]`
    
    이게 있는데, 이 함수는 **extras 테이블을 읽어서 deps로 넣는 로직이 없습니다.**
    
    또한 Poetry의 `groups=["main"]` 같은 정보도 여기서는 별도 처리하지 않습니다.
    
    그래서 **extra 의존성(선택 기능)**은 이 파서 기준으로 `dependenciesList`에 거의 반영되지 않습니다.
    
    ---
    
    ## 7) 이 케이스를 고치려면 어디를 손봐야 하나?
    
    이 `Django` 누락은 “poetry.lock이 잘못”이 아니라, 파서가 **의존성 이름 정규화를 덜 하는 버그/누락**에 가깝다.
    
    가장 간단한 개선은 2차 루프에서 `adep`를 소문자로도 찾아보는 것:
    
    ```jsx
    const k = adep.toLowerCase();
    if (existingPkgMap[k]) depRef = existingPkgMap[k];
    
    ```
    
    그리고 Python 패키지 이름은 PEP 503 스타일로 `[-_.]+`를 `-`로 통일하는 정규화도 많이 씁니다:
    
    ```jsx
    constnorm = (s) => s.toLowerCase().replace(/[-_.]+/g,"-");
    if (existingPkgMap[norm(adep)]) depRef = existingPkgMap[norm(adep)];
    
    ```
    
    이렇게 하면 `Django`도 `django`로 잘 연결됩니다.
    
    ## 1) `pkgList`에서 3개 컴포넌트가 이렇게 생성됨
    
    ```json
    [
    {
    "name":"dj-rest-auth",
    "version":"7.0.1",
    "description":"Authentication and Registration in Django Rest Framework",
    "properties":[
    {
    "name":"cdx:pypi:requiresPython",
    "value":">=3.8"
    }
    ],
    "purl":"pkg:pypi/dj-rest-auth@7.0.1",
    "bom-ref":"pkg:pypi/dj-rest-auth@7.0.1",
    "evidence":{
    "identity":{
    "field":"purl",
    "confidence":1,
    "methods":[
    {
    "technique":"manifest-analysis",
    "confidence":1,
    "value":"/mnt/data/poetry.lock"
    }
    ]
    }
    }
    },
    {
    "name":"django",
    "version":"5.2.6",
    "description":"A high-level Python web framework that encourages rapid development and clean, pragmatic design.",
    "properties":[
    {
    "name":"cdx:pypi:requiresPython",
    "value":">=3.10"
    }
    ],
    "purl":"pkg:pypi/django@5.2.6",
    "bom-ref":"pkg:pypi/django@5.2.6",
    "evidence":{
    "identity":{
    "field":"purl",
    "confidence":1,
    "methods":[
    {
    "technique":"manifest-analysis",
    "confidence":1,
    "value":"/mnt/data/poetry.lock"
    }
    ]
    }
    }
    },
    {
    "name":"djangorestframework",
    "version":"3.16.1",
    "description":"Web APIs for Django, made easy.",
    "properties":[
    {
    "name":"cdx:pypi:requiresPython",
    "value":">=3.9"
    }
    ],
    "purl":"pkg:pypi/djangorestframework@3.16.1",
    "bom-ref":"pkg:pypi/djangorestframework@3.16.1",
    "evidence":{
    "identity":{
    "field":"purl",
    "confidence":1,
    "methods":[
    {
    "technique":"manifest-analysis",
    "confidence":1,
    "value":"/mnt/data/poetry.lock"
    }
    ]
    }
    }
    }
    ]
    
    ```
    
    ### 여기서 “왜 poetry.lock의 files 해시가 안 들어가나?”
    
    네 `poetry.lock`에는 `files = [{file, hash}, ...]`가 있는데, 이 함수는 그걸 읽지 않고 `lockTomlObj.metadata.files[pkg.name]` 형태만 처리하는 분기만 있어서, **현재 코드 그대로면** wheel/sdist 파일 컴포넌트는 만들어지지 않습니다.
    
    ---
    
    ## 2) `dependenciesList`에서 3개 ref 항목이 이렇게 확정됨
    
    아래는 이 함수가 `depsMap`을 2차 패스로 정규화한 뒤, `.sort()`까지 적용해서 넣는 **최종 `dependenciesList` 항목 예시**입니다(요청한 3개만 발췌).
    
    ```json
    [
    {
    "ref":"pkg:pypi/dj-rest-auth@7.0.1",
    "dependsOn":[
    "pkg:pypi/djangorestframework@3.16.1"
    ]
    },
    {
    "ref":"pkg:pypi/djangorestframework@3.16.1",
    "dependsOn":[
    "pkg:pypi/django@5.2.6"
    ]
    },
    {
    "ref":"pkg:pypi/django@5.2.6",
    "dependsOn":[
    "pkg:pypi/asgiref@3.9.1",
    "pkg:pypi/sqlparse@0.5.3",
    "pkg:pypi/tzdata@2025.2"
    ]
    }
    ]
    
    ```
    
    ---
    
    ## 3) dj-rest-auth에서 “django 직접 엣지”가 빠지는 이유
    
    `poetry.lock`에서 dj-rest-auth는 이렇게 되어 있죠:
    
    - `[package.dependencies]`
        - `Django = ">=4.2,<6.0"` ← **대문자 D**
        - `djangorestframework = ">=3.13.0"`
    
    그런데 이 함수는:
    
    - `existingPkgMap`을 만들 때 키를 **무조건 소문자**로 저장하고(`existingPkgMap[name.toLowerCase()] = bomRef`)
    - 의존성 이름을 bom-ref로 바꿀 때는 `existingPkgMap[adep]`처럼 **소문자 정규화를 안 하고 그대로 조회**합니다.
    
    그래서 2차 패스에서
    
    - `"djangorestframework"`(소문자)는 성공 → `pkg:pypi/djangorestframework@3.16.1`
    - `"Django"`(대문자)는 실패 → `pkg:pypi/django@5.2.6`로 못 바뀜
        
        ⇒ 결과적으로 `dj-rest-auth -> django` 직접 엣지가 **누락**됩니다.
        

- cdxgen의 parsePyLockData function에서 existingPkgMap에는 lower 함수로 인해 소문자로 기록 되지만 1차 루프에서 의존성 키를 “그대로” 조회함 → 이 경우 패키지에서 확인한 Django라는 이름을 비교식으로 변환한 django와 1차 루프에서 인증용으로 다시 루프를 돌 때 Django와의 차이점으로 인해 의존성 패키지가 누락 되는 것을 확인할 수 있었음.

```json
// SBOM(cdxgen)에서 dj-rest-auth의 의존성 리스트 부분 발췌
{
      "ref": "pkg:pypi/dj-rest-auth@7.0.1",
      "dependsOn": [
        "pkg:pypi/djangorestframework@3.16.1"
      ]
 }
```

```json
// poetry.lock 파일에서 dj-rest-auth의 패키지 의존성 표현 부분 (Django가 누락된 것을 확인)
[[package]]
name = "dj-rest-auth"
version = "7.0.1"
description = "Authentication and Registration in Django Rest Framework"
optional = false
python-versions = ">=3.8"
groups = ["main"]
files = [
    {file = "dj-rest-auth-7.0.1.tar.gz", hash = "sha256:3f8c744cbcf05355ff4bcbef0c8a63645da38e29a0fdef3c3332d4aced52fb90"},
]

[package.dependencies]
Django = ">=4.2,<6.0"
djangorestframework = ">=3.13.0"

[package.extras]
with-social = ["django-allauth[socialaccount] (>=64.0.0)"]
```