# ITSM Pro

The runnable backend and frontend are pinned under `components/`. To download both in an existing checkout, run `git submodule update --init --recursive`.

Start the backend with `mvn spring-boot:run` from `components/backend`. Its default database is in memory. Start the frontend from `components/frontend` with `npm ci` followed by `npm run dev`. Use `npm run build` for a production frontend build.

