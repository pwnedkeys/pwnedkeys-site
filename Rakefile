task :default => :jekyll

HERE = File.expand_path('..', __FILE__)

desc "Regenerate everything locally"
task :jekyll do
	sh "bundle exec jekyll --url file://#{HERE}/_site --future"
end

desc "Regenerate pages when source content changes"
task :auto do
	sh "bundle exec jekyll --auto --url file://#{HERE}/_site --future"
end

desc "Update the production site"
task :production do |t|
	sh "git push production master"
end
