# Cakefile
# batman
# Copyright Shopify, 2011

muffin       = require 'muffin'
path         = require 'path'
q            = require 'q'
glob         = require 'glob'
{exec, fork, spawn} = require 'child_process'

option '-w', '--watch',  'continue to watch the files and rebuild them when they change'
option '-c', '--commit', 'operate on the git index instead of the working tree'
option '-d', '--dist',   'compile minified versions of the platform dependent code into lib/dist (build task only)'
option '-m', '--compare', 'compare to git refs (stat task only)'
option '-s', '--coverage', 'run jscoverage during tests and report coverage (test task only)'

pipedExec = do ->
  running = false
  pipedExec = (args..., callback) ->
    if !running
      running = true
      child = spawn('node', args, stdio: 'inherit')
      process.on 'exit', exitListener = -> child.kill()
      child.on 'close', (code) ->
        process.removeListener('exit', exitListener)
        running = false
        callback(code)


task 'build', 'compile Batman.js and all the tools', (options) ->
  files = glob.sync('./src/**/*')
  muffin.run
    files: files
    options: options
    map:
      'src/batman\.coffee'            : (matches) -> muffin.compileTree(matches[0], 'lib/batman.js', options)
      'src/platform/([^/]+)\.coffee'     : (matches) -> muffin.compileTree(matches[0], "lib/batman.#{matches[1]}.js", options) unless matches[1] == 'node'
      'src/extras/(.+)\.coffee'       : (matches) -> muffin.compileTree(matches[0], "lib/extras/#{matches[1]}.js", options)
      'tests/run\.coffee'             : (matches) -> muffin.compileTree(matches[0], 'tests/run.js', options)

  invoke 'build:tools'

  if options.dist
    invoke 'build:dist'

task 'build:tools', 'compile command line batman tools and build transforms', (options) ->
  muffin.run
    files: './src/tools/**/*'
    options: options
    map:
      'src/tools/batman\.coffee'      : (matches) -> muffin.compileScript(matches[0], "tools/batman", muffin.extend({}, options, {mode: 0o755, hashbang: true}))
      'src/tools/(.+)\.coffee'        : (matches) -> muffin.compileScript(matches[0], "tools/#{matches[1]}.js", options)

task 'build:dist', 'compile Batman.js files for distribution', (options) ->
  temp    = require 'temp'
  tmpdir = temp.mkdirSync()
  distDir = "lib/dist"
  developmentTransform = require('./tools/remove_development_transform').removeDevelopment

  # Run a task which concats the coffeescript, compiles it, and then minifies it
  first = true
  muffin.run
    files: './src/**/*'
    options: options
    map:
      'src/dist/(.+)\.coffee' : (matches) ->
        return if matches[1] in ['undefine_module']
        destination = "lib/dist/#{matches[1]}.js"
        muffin.compileTree(matches[0], destination).then ->
          options.transform = developmentTransform
          muffin.minifyScript(destination, options).then ->
            muffin.notify(destination, "File #{destination} minified.")

task 'test', ' run the tests continuously on the command line', (options) ->
  pipedExec './node_modules/.bin/karma', 'start', './karma.conf.coffee', (code) ->
    process.exit(code)

task 'test:travis', 'run the tests once using PhantomJS', (options) ->
  pipedExec './node_modules/.bin/karma', 'start', '--single-run', '--browsers', 'PhantomJS', './karma.conf.coffee', (code) ->
    process.exit(code)

task 'stats', 'compile the files and report on their final size', (options) ->
  muffin.statFiles(glob.sync('./src/**/*.coffee').concat(glob.sync('./lib/**/*.js')), options)
