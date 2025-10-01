### Hot Reload와 Watch Mode

- 로컬 개발 시, 소스코드 변경이 즉시 반영되길 원하는 경우가 있음
- NestJS에서는 크게 3가지 방법이 존재함

#### NestJS Builder 종류
- Nest CLI는 내부적으로 빌더(Builder) 를 선택해서 빌드/실행을 진행함
- 선택 방식은 nest-cli.json의 설정 및 설치된 의존성에 따라 달라짐
1. TSC Builder
   - 기본 TypeScript 컴파일러(tsc)를 직접 실행
   - 단순하고 빠르며, watch 시 incremental build 지원
   - 예전 Nest CLI(10 이하) 기본값
2. Webpack Builder
   - webpack, ts-loader를 이용해 빌드
   - 타입체킹과 번들링이 포함되어 안정적이지만 느림
   - Nest CLI 11.x에서는 nest new 시 nest-cli.json에 "webpack": true가 자동 추가되어 기본 선택되는 경우가 있음
   - 실행 로그에 Info Webpack is building your sources... 문구가 출력됨
3. SWC Builder
   - SWC 기반 초고속 빌더
   - 컴파일 속도가 가장 빠르지만, 최신 TS 기능 호환성은 제한적
   - 실험적이므로 선택적으로만 사용

#### Watch Mode & Hot Reload
1. nest start --debug --watch (CLI 자체 watch)
   - 동작 :
     - Nest CLI가 chokidar로 파일 변경 감시 → 빌드 프로세스 실행 → 기존 앱 프로세스를 kill 후 새 프로세스를 fork
   - 빌더 선택 :
     - nest-cli.json에 "webpack": true가 있으면 → Webpack Builder 동작
     - 없으면 → TSC Builder 동작 (내부적으로 nest start --tsc --watch와 동일)
   - 장점 :
     - NestJS 공식 기본 제공
     - CLI가 프로세스를 관리하므로 안정적
   - 단점 : 
     - 항상 앱 전체 재시작 → DB 연결, 세션, 메모리 상태가 매번 초기화 됨
     - Webpack Builder일 경우, 전체 빌드 후 타입체킹까지 하므로 적용 속도가 느림
   ```package.json
   "scripts": {
       "start:debug": "nest start --debug --watch"
   }
   ```

2. nest start --tsc --watch (tsc 직접 watch)
   - 동작 : TypeScript 컴파일러(tsc)의 --watch 모드 실행 → dist 디렉토리 빌드 → Nest CLI가 빌드 이벤트 감지 → Node 프로세스 재시작
   - 장점 :
     - 변경 즉시 빠르게 반영 (tsc는 이벤트 감지를 바로 처리)
     - Webpack 기반 watch보다 훨씬 빠름
   - 단점 :
     - 프로젝트 규모가 크면 전체 빌드 비용이 커짐
   ```package.json
   "scripts": {
       "start:debug": "nest start --tsc --watch"
   }
   ```

3. hot reload (HMR, webpack-hmr.config.js 필요)
   - 동작 :
     - Webpack/Vite 등의 HMR 기능을 활용 → 변경된 모듈만 교체 주입
     - 앱 프로세스는 그대로 유지된 채 특정 모듈만 교체됨
   - 장점 :
     - 앱 전체 재시작이 없으므로 DB 연결, 세션, 메모리 상태 유지
     - 가장 빠른 속도
   - 단점 :
     - 별도 설정 필요 (webpack, webpack-cli, webpack-node-externals 등 설치 + webpack-hmr.config.js 작성)
     - NestJS가 공식 문서로 레시피는 제공하지만 기본 권장은 아님
   ```shell
   $ pnpm add -D webpack webpack-cli webpack-node-externals ts-loader run-script-webpack-plugin
   ```
   ```ts (webpack-hmr.config.js)
   // 프로젝트 루트 폴더에 webpack-hmr.config.js 파일 생성 후 아래 내용 설정
   
   const webpack = require('webpack');
   const nodeExternals = require('webpack-node-externals');
   const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');
   
   module.exports = function (options, webpack) {
       return {
           ...options,
           entry: ['webpack/hot/poll?100', options.entry],
           watch: true,
           externals: [
               nodeExternals({
                   allowlist: ['webpack/hot/poll?100'],
               }),
           ],
           plugins: [
               ...options.plugins,
               new webpack.HotModuleReplacementPlugin(),
               new webpack.WatchIgnorePlugin({
                   paths: [/\.js$/, /\.d\.ts$/],
               }),
               new RunScriptWebpackPlugin({ name: options.output.filename }),
           ],
       };
   };
   ```
   ```ts (main.ts)
   // main.ts 파일 수정
   
   declare const module: any;

   async function bootstrap() {
       const app = await NestFactory.create(AppModule);
       await app.listen(3000);
   
       if (module.hot) {
           module.hot.accept();
           module.hot.dispose(() => app.close());
       }
   }
   bootstrap();
   ```
   ```package.json
   "scripts": {
       "start:hmr": "nest build --webpack --webpackPath webpack-hmr.config.js --watch"
   }
   ```