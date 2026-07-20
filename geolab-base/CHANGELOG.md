# Changelog

## [2.0.0](https://github.com/sparafina-earthscope/Geolab-release-process/compare/v1.1.0...v2.0.0) (2026-07-20)


### ⚠ BREAKING CHANGES

* testing switching base image for second time
* **geolab-base:** base image changed from pangeo/base-image:04bb14b to 04681e6.

### Features

* **geolab-base:** bump base image to pangeo/base-image:04681e6 ([c47b683](https://github.com/sparafina-earthscope/Geolab-release-process/commit/c47b6830e5e35885f4c96e9f02b53c9a57d229c7))
* testing switching base image for second time ([94eee8a](https://github.com/sparafina-earthscope/Geolab-release-process/commit/94eee8a580a75329ae48af413721c5959aed1777))

## [1.1.0](https://github.com/sparafina-earthscope/Geolab-release-process/compare/v1.0.1...v1.1.0) (2026-07-18)


### Features

* **geolab-base:** add nbgitpuller to environment.yml ([831d6c4](https://github.com/sparafina-earthscope/Geolab-release-process/commit/831d6c494131d8e1f5b389c270301ea5e7696f0c))

## [1.0.1](https://github.com/sparafina-earthscope/Geolab-release-process/compare/v1.0.0...v1.0.1) (2026-07-18)


### Bug Fixes

* **geolab-base:** pin seisbench==0.6.0 to test versioning ([67714cc](https://github.com/sparafina-earthscope/Geolab-release-process/commit/67714cc021f29a1f5f4b9ac42f7febf02045d076))

## 1.0.0 (2026-07-17)


### Features

* added seisbench for ML workflows ([9891933](https://github.com/sparafina-earthscope/Geolab-release-process/commit/9891933c4f13c97bd2a7e3e8e420f63862375dd7))
* adding pyrocko package ([e6e3f0f](https://github.com/sparafina-earthscope/Geolab-release-process/commit/e6e3f0fec941413a5f440b315c65a13de764e02c))
* **geolab-base:** add seisbench for seismic ML models ([e794291](https://github.com/sparafina-earthscope/Geolab-release-process/commit/e794291181d9e47f3478ca967b14cd7a2c766ac8))


### Bug Fixes

* reset versioning baseline to 1.0.0 and fix release automation ([543b658](https://github.com/sparafina-earthscope/Geolab-release-process/commit/543b6581756a5a22c2acfbf6252f12153a6ac910))


### Miscellaneous Chores

* pin next release to 1.0.0 ([0a8f81c](https://github.com/sparafina-earthscope/Geolab-release-process/commit/0a8f81ccd17c99b282a8e0e885973a0cd781053a))
* simplify release automation to a basic release-please + build-push setup ([e4487db](https://github.com/sparafina-earthscope/Geolab-release-process/commit/e4487db9810395305d741b2dfd27490749c59f4f))
