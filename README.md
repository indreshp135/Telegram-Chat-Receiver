```js
import React, { useState } from 'react';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { Eye, EyeOff, RefreshCw } from 'lucide-react';

const PasswordInput = () => {
  const [showPassword, setShowPassword] = useState(false);
  const [password, setPassword] = useState('');

  const generatePassword = () => {
    const charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';
    const length = 12;
    let newPassword = '';
    for (let i = 0; i < length; i++) {
      const randomIndex = Math.floor(Math.random() * charset.length);
      newPassword += charset[randomIndex];
    }
    setPassword(newPassword);
  };

  return (
    <div className="relative w-full max-w-md">
      <div className="relative">
        <Input
          type={showPassword ? 'text' : 'password'}
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          className="pr-24"
          placeholder="Enter password"
        />
        <div className="absolute inset-y-0 right-0 flex">
          <Button
            type="button"
            variant="ghost"
            size="sm"
            className="h-full px-2 hover:bg-transparent"
            onClick={() => setShowPassword(!showPassword)}
          >
            {showPassword ? (
              <EyeOff className="h-4 w-4 text-gray-500" />
            ) : (
              <Eye className="h-4 w-4 text-gray-500" />
            )}
          </Button>
          <Button
            type="button"
            variant="ghost"
            size="sm"
            className="h-full px-2 hover:bg-transparent"
            onClick={generatePassword}
          >
            <RefreshCw className="h-4 w-4 text-gray-500" />
          </Button>
        </div>
      </div>
    </div>
  );
};

export default PasswordInput;
```
