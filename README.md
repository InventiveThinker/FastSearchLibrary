# FastSearchLibrary
The multithreading .NET library that provides opportunity to fast find files or directories using different search criteria.
#### Works really fast. Check it yourself!

## INSTALLATION
1. Download archive with last [release](https://github.com/VladPVS/FastSearchLibrary/releases "Last release")
2. Extract content from some directoty.
3. Copy files .dll and .xml files in directory of your project.
4. Add library to your project: Solution Explorer -> Reference -> item AddReference in contex menu -> Browse
5. Add appropriate namespace: `using FastSearchLibrary;`
6. Set target .NET Framework version as 4.5.1 or higher: Project -> <YourProjectName> Properties -> Target framework

## CONTENT

Next classes provide search functionality:
* FileSearcher
* DirectorySearcher
* FileSearcherMultiple
* DirectorySearcherMultiple

## USE PRINCIPLES
### Basic opportunities
  * Classes `FilesSearcher` and `DirectorySearcher` contain static method that allow execute search by different criteria.
  These methods return result only when they fully complete execution.
  * Methods that have "Fast" ending divide task on several 
  subtasks that executes simultaneously in thread pool.
  * Methods that have "Async" ending return Task and don't block the called thread.
  * First group of methods accept 2 parameters: 
    * `string folder` - start search directory
    * `string pattern` - the search string to match against the names of files in path.
    This parameter can contain a combination of valid literal path and wildcard (* and ?)
    characters, but doesn't support regular expressions.
    
  Examples:
  
    List<FileInfo> files = FileSearcher.GetFiles(@"C:\Users", "*.txt");
   Finds all `*.txt` files in `C:\Users` using one thread method.
   
    List<FileInfo> files = FileSearcher.GetFilesFast(@"C:\Users", "*SomePattern*.txt");
   Finds all files that match appropriate pattern using several thread in thread pool.
   
    Task<List<FileInfo>> task = FileSearcher.GetFilesFastAsync(@"C:\", "a?.txt");
   Finds all files that match appropriate pattern using several thread in thread pool as
   an asynchronous operation.
   
   * Second group of methods accept 2 parameters:
     * `string folder` - start search directory
     * `Func<FileInfo, bool> isValid` - delegate that determines algorithm of file selection.
     
   Examples:
   
    Task<List<FileInfo>> task = FileSearcher.GetFilesFastAsync(@"D:\", (f) =>
    {
         return (f.Name.Contains("Pattern") || f.Name.Contains("Pattern2")) &&
                 f.LastAccessTime >= new DateTime(2018, 3, 1) && f.Length > 1073741824;
    });
   Finds all files that match appropriate conditions using several thread in thread pool as
   an asynchronous operation.
   
   You also can use regular expressions:
    
    Task<List<FileInfo>> task = FileSearcher.GetFilesFastAsync(@"D:\", (f) =>
    {
         return (f) => Regex.IsMatch(f.Name, @".*Imagine[\s_-]Dragons.*.mp3$");
    }); 
    
   Finds all files that match appropriate regular expression using several thread in thread pool as
   an asynchronous operation.
   
 ### Advanced opportunities
   If you want to execute some complicated search with realtime result getting you should use instance of `FileSearcher` class,
   that has various constructor overloads.
   `FileSearcher` class includes next events:
   * `event EventHandler<FileEventArgs> FilesFound` - fires when next portion of files is found.
     Event includes `List<FileInfo> Files { get; }` property that contains list of finding files.
   * `event EventHandler<SearchCompleted> SearchCompleted` - fires when search process is completed or stopped. 
     Event includes `bool IsCanceled { get; }` property that contains value that defines whether search procees stopped by calling
     `StopSearch()` method. 
    To stop search process possibility one have to use constructor that accept CancellationTokenSource parameter.
    
    Example:
    
    class Searcher
    {
        private static object locker = new object(); // locker object

        private FileSearcher searcher;

        List<FileInfo> files;

        public Searcher()
        {
            files = new List<FileInfo>(); // create list that will contain search result
        }

        public void Startsearch()
        {
            CancellationTokenSource tokenSource = new CancellationTokenSource();
            // create tokenSource to get stop search process possibility

            searcher = new FileSearcher(@"C:\", (f) =>
            {
               return Regex.IsMatch(f.Name, @".*[iI]magine[\s_-][dD]ragons.*.mp3$"); 
            }, tokenSource);  // give tokenSource in constructor
 

            searcher.FilesFound += (sender, arg) => // subscribe on FilesFound event
            {
                lock (locker) // using a lock is obligatorily
                {
                    arg.Files.ForEach((f) =>
                    {
                        files.Add(f); // add the next part of the received files to the results list
                        Console.WriteLine($"File location: {f.FullName}, \nCreation.Time: {f.CreationTime}");
                    });

                    if (files.Count >= 10) // one can choose any stopping condition
                       searcher.StopSearch();
                }
            };

            searcher.SearchCompleted += (sender, arg) => // subscribe on SearchCompleted event
            {
                if (arg.IsCanceled) // check whether StopSearch() called
                    Console.WriteLine("Search stopped.");
                else
                    Console.WriteLine("Search completed.");

                Console.WriteLine($"Quantity of files: {files.Count}"); // show amount of finding files
            };

            searcher.StartSearchAsync();
            // start search process as an asynchronous operation that doesn't block the called thread
        }
    }
 Note that all `FilesFound` event handlers are not thread safe so to prevent result loosing one should use
 `lock` keyword as you can see in example above or use thread safe collection from `System.Collections.Concurrent` namespace.
 
 ### Extended opportunities
   FileSearcher class includes 2 additional parameters: `handlerOption`, `suppressOperationCanceledException`.
   `ExecuteHandlers handlerOption` parameter represents instance of `ExecuteHandlers` enumeration that specifies where
   FilesFound event handlers are executed:  
   * `InCurrentTask` value means that `FileFound` event handlers will be executed in that task where files were found. 
   * `InNewTask` value means that `FilesFound` event handlers will be executed in new task.
    Default value is `InCurrentTask`. It is more preferably in most cases. `InNewTask` value one should use only if handlers execute
    very sophisticated work that takes a lot of time, e.g. parsing of each found file.
    
   `bool suppressOperationCanceledException` parameter determines whether necessary to suppress 
   OperationCanceledException.
   If `suppressOperationCanceledException` parameter has value `false` and StopSearch() method calls the `OperationCanceledException` 
   will be thrown. In this case you have to process the exception manually.
   If `suppressOperationCanceledException` parameter has value `true` and StopSearch() method calls the `OperationCanceledException` 
   is processed automatically and you don't need to catch it. 
   Default value is `true`.
   
   Example:
            
    CancellationTokenSource tokenSource = new CancellationTokenSource();

    FileSearcher searcher = new FileSearcher(@"D:\Program Files", (f) =>
    {
       return Regex.IsMatch(f.Name, @".{1,5}[Ss]ome[Pp]attern.txt$") && (f.Length >= 8192); // 8192b == 8Kb 
    }, tokenSource, ExecuteHandlers.InNewTask, true); // suppressOperationCanceledException == true
    
   ### MULTIPLE SEARCH
   `FileSearcher` and `DirectorySearcher` classes can search only in one directory (and in all subdirectories surely) 
   but what if you want to perform search in several directories at same time?     
   Of course, you can create some instances of `FileSearcher` (or `DirectorySearcher`) class and launch them simultaneously, 
   but `FilesFound` (or `DirectoriesFound`) events will occur for each instance you create. As a rule, it's inconveniently.
   Classes `FileSearcherMultiple` and `DirectorySearcherMultiple` are intended to solve this problem. 
   They are similar to `FileSearcher` and `DirectorySearcher` but can execute search in several directories.
   The difference between `FileSearcher` and `FileSearcheMultiple` is that constructor of `Multiple` class accepts list of 
   directories instead one directory.
   
   Example:
   
    List<string> folders = new List<string>
    {
      @"C:\Users\Public",
      @"C:\Windows\System32",
      @"D:\Program Files",
      @"D:\Program Files (x86)"
    }; // list of search directories

    List<string> keywords = new List<string> { "word1", "word2", "word3" }; // list of search keywords

    FileSearcherMultiple multipleSearcher = new FileSearcherMultiple(folders, (f) =>
    {
      if (f.CreationTime >= new DateTime(2015, 3, 15) &&
         (f.Extension == ".cs" || f.Extension == ".sln"))
        foreach (var keyword in keywords)
          if (f.Name.Contains(keyword))
            return true;
      return false;
    }, tokenSource, ExecuteHandlers.InCurrentTask, true);       

### SPEED OF WORK
It depends on your computer performance, current loading, but usually `Fast` methods and instance method `StartSearch()` are
performed at least in 2 times faster then simple one-thread recursive algorithm if you use modern multicore processor of course.    
   
