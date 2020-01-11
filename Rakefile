require "rake/testtask"

require File.expand_path(File.dirname(__FILE__)) + "/test/config"
require File.expand_path(File.dirname(__FILE__)) + "/test/support/config"

def run_without_aborting(*tasks)
  errors = []

  tasks.each do |task|
    begin
      Rake::Task[task].invoke
    rescue Exception
      errors << task
    end
  end

  abort "Errors running #{errors.join(', ')}" if errors.any?
end

desc "Run mysql2, sqlite, and postgresql tests by default"
task default: :test

task :package

desc "Run mysql2, sqlite, and postgresql tests"
task :test do
  tasks = defined?(JRUBY_VERSION) ?
    %w(test_jdbcmysql test_jdbcsqlite3 test_jdbcpostgresql) :
    %w(test_mysql2 test_sqlite3 test_postgresql)
  run_without_aborting(*tasks)
end

namespace :test do
  task :isolated do
    tasks = defined?(JRUBY_VERSION) ?
      %w(isolated_test_jdbcmysql isolated_test_jdbcsqlite3 isolated_test_jdbcpostgresql) :
      %w(isolated_test_mysql2 isolated_test_sqlite3 isolated_test_postgresql)
    run_without_aborting(*tasks)
  end
end

desc "Build MySQL and PostgreSQL test databases"
namespace :db do
  task create: ["db:mysql:build", "db:postgresql:build"]
  task drop: ["db:mysql:drop", "db:postgresql:drop"]
end

%w( mysql2 postgresql sqlite3 sqlite3_mem db2 oracle jdbcmysql jdbcpostgresql jdbcsqlite3 jdbcderby jdbch2 jdbchsqldb ).each do |adapter|
  namespace :test do
    Rake::TestTask.new(adapter => "#{adapter}:env") { |t|
      adapter_short = adapter == "db2" ? adapter : adapter[/^[a-z0-9]+/]
      t.libs << "test"
      t.test_files = (Dir.glob("test/cases/**/*_test.rb").reject {
        |x| x.include?("/adapters/")
      } + Dir.glob("test/cases/adapters/#{adapter_short}/**/*_test.rb"))

      t.warning = true
      t.verbose = true
      t.ruby_opts = ["--dev"] if defined?(JRUBY_VERSION)
    }

    namespace :isolated do
      task adapter => "#{adapter}:env" do
        adapter_short = adapter == "db2" ? adapter : adapter[/^[a-z0-9]+/]
        puts [adapter, adapter_short].inspect

        dash_i = [
          "test",
          "lib",
          "../activesupport/lib",
          "../activemodel/lib"
        ].map { |dir| File.expand_path(dir, __dir__) }

        dash_i.reverse_each do |x|
          $:.unshift(x) unless $:.include?(x)
        end
        $-w = true

        require "bundler/setup" unless defined?(Bundler)

        # Every test file loads "cases/helper" first, so doing it
        # post-fork gains us nothing.

        # We need to dance around minitest autorun, though.
        require "minitest"
        Minitest.instance_eval do
          alias _original_autorun autorun
          def autorun
            # no-op
          end
          require "cases/helper"
          alias autorun _original_autorun
        end

        failing_files = []

        test_options = ENV["TESTOPTS"].to_s.split(/[\s]+/)

        test_files = (Dir["test/cases/**/*_test.rb"].reject {
          |x| x.include?("/adapters/")
        } + Dir["test/cases/adapters/#{adapter_short}/**/*_test.rb"]).sort

        if ENV["BUILDKITE_PARALLEL_JOB_COUNT"]
          n = ENV["BUILDKITE_PARALLEL_JOB"].to_i
          m = ENV["BUILDKITE_PARALLEL_JOB_COUNT"].to_i

          test_files = test_files.each_slice(m).map { |slice| slice[n] }.compact
        end

        test_files.each do |file|
          puts "--- #{file}"
          fake_command = Shellwords.join([
            FileUtils::RUBY,
            "-w",
            *dash_i.map { |dir| "-I#{Pathname.new(dir).relative_path_from(Pathname.pwd)}" },
            file,
          ])
          puts fake_command

          # We could run these in parallel, but pretty much all of the
          # railties tests already run in parallel, so ¯\_(⊙︿⊙)_/¯
          Process.waitpid fork {
            ARGV.clear.concat test_options
            Rake.application = nil

            Minitest.autorun

            load file
          }

          unless $?.success?
            failing_files << file
            puts "^^^ +++"
          end
          puts
        end

        puts "--- All tests completed"
        unless failing_files.empty?
          puts "^^^ +++"
          puts
          puts "Failed in:"
          failing_files.each do |file|
            puts "  #{file}"
          end
          puts

          exit 1
        end
      end
    end
  end

  namespace adapter do
    task test: "test_#{adapter}"
    task isolated_test: "isolated_test_#{adapter}"

    # Set the connection environment for the adapter
    task(:env) { ENV["ARCONN"] = adapter }
  end

  # Make sure the adapter test evaluates the env setting task
  task "test_#{adapter}" => ["#{adapter}:env", "test:#{adapter}"]
  task "isolated_test_#{adapter}" => ["#{adapter}:env", "test:isolated:#{adapter}"]
end

namespace :db do
  namespace :mysql do
    connection_arguments = lambda do |connection_name|
      config = ARTest.config["connections"]["mysql2"][connection_name]
      ["--user=#{config["username"]}", "--password=#{config["password"]}", ("--host=#{config["host"]}" if config["host"])].join(" ")
    end

    desc "Build the MySQL test databases"
    task :build do
      config = ARTest.config["connections"]["mysql2"]
      %x( mysql #{connection_arguments["arunit"]} -e "create DATABASE #{config["arunit"]["database"]} DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_unicode_ci ")
      %x( mysql #{connection_arguments["arunit2"]} -e "create DATABASE #{config["arunit2"]["database"]} DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_unicode_ci ")
    end

    desc "Drop the MySQL test databases"
    task :drop do
      config = ARTest.config["connections"]["mysql2"]
      %x( mysqladmin #{connection_arguments["arunit"]} -f drop #{config["arunit"]["database"]} )
      %x( mysqladmin #{connection_arguments["arunit2"]} -f drop #{config["arunit2"]["database"]} )
    end

    desc "Rebuild the MySQL test databases"
    task rebuild: [:drop, :build]
  end

  namespace :postgresql do
    desc "Build the PostgreSQL test databases"
    task :build do
      config = ARTest.config["connections"]["postgresql"]
      %x( createdb -E UTF8 -T template0 #{config["arunit"]["database"]} )
      %x( createdb -E UTF8 -T template0 #{config["arunit2"]["database"]} )
    end

    desc "Drop the PostgreSQL test databases"
    task :drop do
      config = ARTest.config["connections"]["postgresql"]
      %x( dropdb #{config["arunit"]["database"]} )
      %x( dropdb #{config["arunit2"]["database"]} )
    end

    desc "Rebuild the PostgreSQL test databases"
    task rebuild: [:drop, :build]
  end
end

task build_mysql_databases: "db:mysql:build"
task drop_mysql_databases: "db:mysql:drop"
task rebuild_mysql_databases: "db:mysql:rebuild"

task build_postgresql_databases: "db:postgresql:build"
task drop_postgresql_databases: "db:postgresql:drop"
task rebuild_postgresql_databases: "db:postgresql:rebuild"

task :lines do
  load File.expand_path("..", File.dirname(__FILE__)) + "/tools/line_statistics"
  files = FileList["lib/active_record/**/*.rb"]
  CodeTools::LineStatistics.new(files).print_loc
end