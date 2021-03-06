#  RubyFuse API

## A Warning

If you use DirLink (in demo.rb) or in any way access a RubyFuse filesystem from *within* the ruby script accessing the RubyFuse, then RubyFuse will hang, and the only recourse is a kill -KILL.

Also: If there are any open files or shells with 'pwd's in your filesystem when you exit your ruby script, fuse *might* not actually be unmounted. To unmount a path yourself, run the command:

		$ fusermount -u <path>

to unmount any FUSE filesystems mounted at <path>.

## The API

RubyFuse provides a layer of abstraction to a programmer who wants to create a virtual filesystem via FUSE.

RubyFuse programs consist of two parts:

1. 	RubyFuse, which is defined in 'RubyFuse.rb'
2.	An object that defines a virtual directory. This must define a number of methods (given below, in "Directory Methods" section) in order to be usable.

To write a RubyFuse program, you must:

1.	Define and create a Directory object that responds to the methods required by RubyFuse for its desired use.

2.	Call RubyFuse.set_root <virtualdir> with the object defining your virtual
		directory.

3.	Mount RubyFuse under a real directory on your filesystem.

4.	Call RubyFuse.run to start receiving and executing events.

Happy Filesystem Hacking!

## 	Hello World Example

### helloworld.rb

This creates a filesystem that contains exactly 1 file: "hello.txt" that, when read, returns "Hello, World!"

This is not writable, and contains no other files.

		require 'RubyFuse'

		class HelloDir
		  def contents(path)
		    ['hello.txt']
		  end
		  def file?(path)
		    path -- '/hello.txt'
		  end
		  def read_file(path)
		    "Hello, World!\n"
		  end
		end

		hellodir = HelloDir.new
		RubyFuse.set_root( hellodir )

		# Mount under a directory given on the command line.
		RubyFuse.mount_under ARGV.shift
		RubyFuse.run

## Directory Methods

Without any methods defined, any object is by default a content-less, file-less directory.

The following are necessary for most or all filesystems:

  Directory listing and file type methods:

    :contents(path)     # Return an array of file and dirnames within <path>.
    :directory?(path)   # Return true if <path> is a directory.
    :file?(path)        # Return true if <path> is a file (not a directory).
    :executable?(path)  # Return true if <path> is an executable file.
    :size(path)         # Return the file size. Necessary for apache, xmms,
                          etc.

  File reading:

    :read_file(path)    # Return the contents of the file at location <path>.

The following are only necessary if you want a filesystem that can be modified
by the user. Without defining any of the below, the contents of the filesystem
are automatically read-only.

  File manipulation:

    :can_write?(path)   # Return true if the user can write to file at <path>.
    :write_to(path,str) # Write the contents of <str> to file at <path>.

    :can_delete?(path)  # Return true if the user can delete file at <path>.
    :delete(path)       # Delete the file at <path>

  Directory manipulation:

    :can_mkdir?(path)   # Return true if user can make a directory at <path>.
    :mkdir(path)        # Make a directory at path.

    :can_rmdir?(path)   # Return true if user can remove directory at <path>.
    :rmdir(path)        # Remove it.

  Neat "toy":

    :touch(path)        # Called when a file is 'touch'd or otherwise has
                          their timestamps explicitly modified. I envision
                          this as a neat toy, maybe you can use it for a
                          push-button file?
                            "touch button" -> unmounts RubyFuse?
                            "touch musicfile.mp3" -> Play the mp3.

If you want a lower level control of your file, then you can use:

    :raw_open(path,mode)   # mode is "r" "w" or "rw", with "a" if the file
                             is opened for append. If raw_open returns true,
                             then the following calls are made:
    :raw_read(path,off,sz) # Read sz bites from file at path starting at
                             offset off
    :raw_write(path,off,sz,buf) # Write sz bites of buf to path starting at
                                  offset off
    :raw_close(path)       # Close the file.
    

## Method call flow

List contents:
		:directory? will be checked before :contents
(Most 'ls' or 'dir' functions will go on next to getattr() for all contents)

Read file:
		:file? will be checked before :read_file

Getattr
		:directory? will be checked first.

  	:file? will be checked before :can_write?
  	:file? will be checked before :executable?

Writing files:
  * directory? is usually called on the directory
    The FS wants to write a new file to, before this
    can occur.
			:can_write? will be checked before :write_to

Deleting files:
	  :file? will be checked before :can_delete?
	  :can_delete? will be checked before :delete

Creating dirs:
  * directory? is usually called on the directory
    The FS wants to make a new directory in, before this can occur.
	  :directory? will be checked.
	  :can_mkdir? is called only if :directory? is false.
	  :can_mkdir? will be checked before :mkdir

Deleting dirs:
	  :directory? will be checked before :can_rmdir?
	  :can_rmdir? will be checked before :rmdir


## module RubyFuse


RubyFuse methods:
  
  RubyFuse.set_root(object)
      Set the root virtual directory to <object>. All queries for obtaining
      file information is directed at object.
  
  RubyFuse.mount_under(path[,opt[,opt,...]])
      This will cause RubyFuse to virtually mount itself under the given path.
      'path' is required to be a valid directory in your actual filesystem.

      'opt's are FUSE options. Most likely, you will only want 'allow_other'
      or 'allow_root'. The two are mutually exclusive in FUSE, but allow_other
      will let other users, including root, access your filesystem. allow_root
      will only allow root to access it.
      
      Also available for RubyFuse users are:
        default_permissions, max_read=N, fsname=NAME.

      For more information, look at FUSE.

      (P.S: I know FUSE allows other options, but I don't think any of the
      rest will do any good with RubyFuse. If you think otherwise, please let me
      know!)

  RubyFuse.run
      This is the final step to make your virtual filesystem accessible. It is
      recommended you run this as your main thread, but you can thread off to
      run this.

  RubyFuse.handle_editor = bool (true by default)
      If handle_editor is true, then RubyFuse will attempt to capture all editor
      files and prevent them from being passed to FuseRoot. It also prevents
      created and unmodified files from being passed as well, as vim (among
      others) will attempt to create and then remove a file that does not
      exist.

  RubyFuse.reader_uid and RubyFuse.reader_gid
      When the filesystem is accessed, the accessor's uid or gid is returned
      by RubyFuse.reader_uid and RubyFuse.reader_gid. You can use this in
      determining your permissions, or even provide different files for
      different users!
  
  RubyFuse.fuse_fd and RubyFuse.process
      These are not intended for use by the programmer. If you want to muck
      with this, read the code to see what they do :D.



FuseDir
----------

RubyFuse::FuseDir defines the methods "split_path" and "scan_path". You
should typically inherit from FuseDir in your own directory objects.

	  base, rest = split_path(path)   # base is the file or directory in the
	                                    current context. rest is either nil,
	                                    or a path that is requested. If 'rest'
	                                    exists, then you should recurse the paths.
	  base, *rest = scan_path(path)   # scan_path returns an array of all
	                                    directory and file elements given by
	                                    <path>. This is useful when you're
	                                    encapsulating an entire fs into one
	                                    object.

MetaDir
-------

MetaDir is a full filesystem defined with hashes. It is writable, and the user
can create and edit files within it, as well as the programmer.

Usage:
	  root = MetaDir.new

	  root.mkdir("/hello")
	  root.write_to("/hello/world","Hello, World!\n")
	  root.write_to("/hello/everybody","Hello, Everybody!\n")

	  RubyFuse.set_root(root)

Because MetaDir is fully recursive, you can mount your own or other defined
directory structures under it. For example, to mount a dictionary filesystem
(as demonstrated in samples/dictfs.rb), use:

	  root.mkdir("/dict",DictFS.new)

Conclusion
----------

Happy Hacking! If you do anything neat with this, please let me know!

Thanks for using RubyFuse!
