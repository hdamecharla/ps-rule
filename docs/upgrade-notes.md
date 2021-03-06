# Upgrade notes

This document contains notes to help upgrade from previous versions of the PSRule GitHub Action.

## Upgrade to v0.3.0 from v0.2.x

Follow these notes to upgrade from action version _v0.2.x_ to _v0.3.0_.
For a list of upstream changes to the PSRule engine see [change log](https://github.com/microsoft/PSRule/blob/main/CHANGELOG.md#v0200).

## Repository analysis

Previously in _v0.2.0_ or prior using `inputType: repository` would:

- Emit a `System.IO.DirectoryInfo` object for the repository root.
- Emit a `System.IO.FileInfo` object for each file in the repository.

This allows rules to be run for the repository and each repository file.

For example:

```powershell
# Synopsis: Check for recommended community files
Rule 'GitHub.Community' -Type 'System.IO.DirectoryInfo' {
    $requiredFiles = @(
        'README.md'
        'LICENSE'
        'CODE_OF_CONDUCT.md'
        'CONTRIBUTING.md'
        '.github/CODEOWNERS'
        '.github/PULL_REQUEST_TEMPLATE.md'
    )
    Test-Path -Path $TargetObject.FullName;
    for ($i = 0; $i -lt $requiredFiles.Length; $i++) {
        $filePath = Join-Path -Path $TargetObject.FullName -ChildPath $requiredFiles[$i];
        $Assert.Create((Test-Path -Path $filePath -PathType Leaf), "$($requiredFiles[$i]) does not exist");
    }
}

# Synopsis: Check for license in code files
Rule 'OpenSource.License' -Type 'System.IO.FileInfo' -If { $TargetObject.Extension -in '.cs', '.ps1', '.psd1', '.psm1' } {
    $commentPrefix = "`# ";
    if ($TargetObject.Extension -eq '.cs') {
        $commentPrefix = '// '
    }
    $header = GetLicenseHeader -CommentPrefix $commentPrefix;
    $content = Get-Content -Path $TargetObject.FullName -Raw;
    $content.StartsWith($header);
}
```

In _v0.3.0_, repository analysis switched to using built-in `File` input type support.
This change provides a more efficient scan of files and allows exclusion of files by path spec.

Two breaking changes occurred as a result of this.

1. The repository root uses the type `PSRule.Data.RepositoryInfo` instead of `System.IO.DirectoryInfo`.
Rules that previously used `-Type 'System.IO.DirectoryInfo'` should now use `-Type 'PSRule.Data.RepositoryInfo'`.
2. Each repository file now uses the type `PSRule.Data.InputFileInfo` instead of `System.IO.FileInfo`.
Additionally, PSRule automatically uses the file extension for type binding.
Rules that previously used `-Type 'System.IO.FileInfo'` should now use the file extension.
i.e. `-Type '.ps1', '.cs', '.js'`.

For example:

```powershell
# Synopsis: Check for recommended community files
Rule 'OpenSource.Community' -Type 'PSRule.Data.RepositoryInfo' {
    $Assert.FilePath($TargetObject, 'FullName', @('CHANGELOG.md'));
    $Assert.FilePath($TargetObject, 'FullName', @('LICENSE'));
    $Assert.FilePath($TargetObject, 'FullName', @('CODE_OF_CONDUCT.md'));
    $Assert.FilePath($TargetObject, 'FullName', @('CONTRIBUTING.md'));
    $Assert.FilePath($TargetObject, 'FullName', @('SECURITY.md'));
    $Assert.FilePath($TargetObject, 'FullName', @('README.md'));
    $Assert.FilePath($TargetObject, 'FullName', @('.github/CODEOWNERS'));
    $Assert.FilePath($TargetObject, 'FullName', @('.github/PULL_REQUEST_TEMPLATE.md'));
}

# Synopsis: Check for license in code files
Rule 'OpenSource.License' -Type '.cs', '.ps1', '.psd1', '.psm1' {
    $Assert.FileHeader($TargetObject, 'FullName', @(
        'Copyright (c) Microsoft Corporation.'
        'Licensed under the MIT License.'
    ));
}
```

> To read more about the new `FilePath` and `FileHeader` assertion helpers see [about_PSRule_Assert]

[about_PSRule_Assert]: https://microsoft.github.io/PSRule/concepts/PSRule/en-US/about_PSRule_Assert.html
